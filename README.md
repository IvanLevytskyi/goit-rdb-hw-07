# HW7 — Time-series та JSONB pipeline у PostgreSQL

Відтворюваний pipeline для двох типів даних, які типово використовуються для ML-ознак:

1. **Time-series:** денні агрегати поїздок NYC Yellow Taxi по pickup-зонах і window-function ознаки для прогнозування попиту.
2. **Event-log:** синтетичний напівструктурований лог подій користувачів у `JSONB`, індексація GIN / B-tree і порівняння з нормалізованою таблицею.
---

## Джерела даних

### NYC TLC Yellow Taxi Trip Records

- **Джерело:** NYC Taxi & Limousine Commission — <https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page>
- **Файл:** одна місячна партиція у форматі Parquet, `yellow_tripdata_2026-08.parquet`
  <https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2026-08.parquet>
- **Ліцензія / умови:** публічні відкриті дані NYC Open Data.

Використовуються лише 7 колонок:

| Колонка | Тип у raw-даних | Використання |
|---|---|---|
| `tpep_pickup_datetime` | timestamp, локальний NYC wall-clock | день поїздки (`ts_day`) |
| `tpep_dropoff_datetime` | timestamp, локальний NYC wall-clock | контроль тривалості |
| `PULocationID` | integer | pickup-зона — ключ часового ряду |
| `DOLocationID` | integer | dropoff-зона |
| `trip_distance` | numeric | дистанція, милі |
| `fare_amount` | numeric | базова ціна поїздки |
| `passenger_count` | numeric | кількість пасажирів |

## Як отримуються файли

Після завантаження `yellow_tripdata_2026-08.parquet`:

1. Читаються лише потрібні колонки і не більше `MAX_ROWS = 500_000` рядків. Читання йде батчами через `pyarrow.ParquetFile.iter_batches`, щоб не тримати весь місяць (~3 млн рядків) у пам'яті.
2. Notebook показує, скільки рядків відсікає кожне правило очищення.
3. Базове очищення від outliers:
   - `0 < fare_amount < 500`
   - `0 < trip_distance < 100`
   - `PULocationID IS NOT NULL`
4. Дані записуються в PostgreSQL (`df.to_sql('nyc_taxi_raw', ...)`, `chunksize=10_000`).

Файли зберігаються в `/content` (Colab) або в поточній директорії. Дані PostgreSQL лежать у `/tmp/hw7_pg`.

---

## Time-series features

Усі ознаки рахуються на `t7_hw_taxi_daily` через window functions. **Кожне вікно має `PARTITION BY pickup_zone_id`**: кожна зона — окремий часовий ряд. Notebook перевіряє це автоматично. Він дістає збережене визначення view через `pg_get_viewdef` і падає з помилкою, якщо хоч одне вікно не партиціоноване.

| Категорія | Ознака | Вираз | Вікно |
|---|---|---|---|
| Lag | `trips_lag_1d`, `trips_lag_7d`, `trips_lag_30d` | `LAG(trip_count, k)` | `ORDER BY ts_day` |
| Rolling mean | `rolling_avg_fare_7d` | `AVG(avg_fare)` | `RANGE BETWEEN INTERVAL '6 days' PRECEDING AND CURRENT ROW` |
| Rolling volatility | `rolling_std_fare_30d` | `STDDEV_SAMP(avg_fare)` | `RANGE BETWEEN INTERVAL '29 days' PRECEDING AND CURRENT ROW` |
| Expanding | `cumulative_trips`, `expanding_avg_trips` | `SUM` / `AVG(trip_count)` | `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` |

View `v_t7_hw_taxi_features` залишає тільки рядки з повною 30-денною історією (`WHERE trips_lag_30d IS NOT NULL`).

**Що ще показує notebook:**

- **Leakage без `PARTITION BY`.** Демонстрація, як `LAG` на першому дні зони бере значення з попередньої зони. Це «тиха» помилка: SQL не видає жодного попередження.
- **Row-based vs calendar lag.** `LAG(trip_count, 7)` рахує рядки, а не дні. Якщо в зоні пропущено день, lag дивиться далі в минуле, ніж обіцяє назва ознаки. Notebook рахує такі випадки і показує календарно точну альтернативу через `LEFT JOIN` на `ts_day - 7`. `RANGE ... INTERVAL` у rolling-ознаках цієї проблеми не має.
- **Target leakage.** Rolling-вікна включають поточний день (`CURRENT ROW`). Якщо модель прогнозує день `D` заздалегідь, вікна треба зсунути на `1 PRECEDING`.

---

## JSONB-частина

### Event-log (Завдання 3)

| `event_type` | ключі `payload` |
|---|---|
| `page_view` | `url`, `referrer`, `device` |
| `click` | `element_id`, `page_url`, `device` |
| `add_to_cart` | `product_id`, `price`, `quantity`, `tags` (масив) |
| `checkout` | `order_id`, `total`, `items_count`, `payment_method` |
| `recommendation_shown` | `model_name`, `model_version`, `recommendations` (масив), `experiment` |

Різну структуру `payload` для різних типів підтверджує запит через `jsonb_object_keys` + `jsonb_typeof`.

Продемонстровані оператори:

- **Containment `@>`** — пошук за парою ключ–значення.
- **Key existence `?`, `?|`, `?&`** — наявність top-level ключів. Notebook показує, що вони не бачать елементів масивів.
- **JSONPath** `jsonb_path_query`, `jsonb_path_exists`, `@?` — розгортання масивів і фільтри всередині документа.

Для кожного оператора є контрольний запит, який порівнює результат з еквівалентним записом через `->>`.

### Індекси та EXPLAIN ANALYZE (Завдання 4)

Обов'язкові `EXPLAIN (ANALYZE, BUFFERS)`:

1. `@>` до GIN;
2. `@>` після GIN `jsonb_path_ops`;
3. `payload ->> 'device' = 'mobile'` без expression index (GIN уже є, але не допомагає);
4. той самий запит після expression B-tree.

---

## Запуск

### Google Colab (рекомендовано)

1. Відкрийте <https://colab.research.google.com> → **File → Upload notebook** → `goit_rdb_hw_07.ipynb`.
2. **Runtime → Run all.**
4. Збережіть notebook з output-ами: **File → Download → .ipynb**. Переконайтеся, що в **Edit → Notebook settings** не ввімкнено *Omit code cell output when saving this notebook*, інакше output-и `EXPLAIN` не збережуться.

### Важливо щодо порядку виконання

- Запускайте notebook **зверху вниз** (Run all).
- Повторний повний запуск безпечний: усі таблиці створюються через `DROP ... IF EXISTS` / `CREATE`.
