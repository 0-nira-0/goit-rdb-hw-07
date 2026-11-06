# goit-rdb-hw-07

Домашнє завдання 7 курсу «Relational Databases for Data Science»: time-series ознаки на NYC Taxi та JSONB event-log з індексами GIN і expression B-tree у PostgreSQL 16.

## Дані

- Джерело: [NYC TLC Trip Record Data](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page), Yellow Taxi.
- Файли: `yellow_tripdata_2024-02.parquet`, `-03` з `https://d37ci6vzurychx.cloudfront.net/trip-data/`. Notebook завантажує їх сам, у репозиторії даних немає.
- Використано варіант A, stub-schema fallback не використовується.

### Формування вибірки

Умова пропонує перші 500 000 рядків березня, але цей зріз закінчується 6 березня, тож `trips_lag_30d` завжди NULL, а view порожній. Тому масштаб лишено тим самим, а зріз розтягнуто на 60 днів: із лютого і березня 2024 береться випадкова вибірка з однаковою часткою 7.6% (`random_state=42`). Файли читаються по одному, у пам'яті лишається тільки вибірка. `trip_count` тому означає кількість поїздок у вибірці, а не повний підрахунок.

| Крок | Рядків |
| --- | --- |
| Два файли разом | 6 590 154 |
| Вибірка 7.6% | 500 852 |
| Після правил `0 < fare_amount < 500`, `0 < trip_distance < 100` | 481 863 |
| Після фільтра посадки в межах 2024-02-01 .. 2024-03-31 (`nyc_taxi_raw`) | 481 862 |
| `t7_hw_taxi_daily` (zone × day) | 8 487 |
| `v_t7_hw_taxi_features` (240 зон × 30 днів) | 7 200 |

## Time-series features

Усі вікна мають `PARTITION BY pickup_zone_id`.

- Lag: `trips_lag_1d`, `trips_lag_7d`, `trips_lag_30d`.
- Rolling average: `rolling_avg_fare_7d`, frame `RANGE BETWEEN INTERVAL '6 days' PRECEDING AND CURRENT ROW`.
- Volatility: `rolling_std_fare_30d`, `STDDEV_SAMP` за 30 календарних днів.
- Expanding: `cumulative_trips`, `expanding_avg_trips`.

`LAG` рахує рядки, а не дні, тоді як 175 із 240 зон мають дні без рядка. Тому view `v_t7_hw_taxi_features` спершу доповнює кожну зону повним календарем (`trip_count = 0`, `avg_fare` NULL), а назви колонок лишаються як в умові. Ознаки звірені з незалежним розрахунком у pandas.

## JSONB-частина

- `t7_hw_events`: 50 000 синтетичних подій, 5 типів із різними схемами payload.
- Запити: containment `@>`, key existence `?` і `?|`, JSONPath (`jsonb_path_query`, `jsonb_path_exists`, `@?`).
- Індекси: GIN `jsonb_path_ops`, GIN `jsonb_ops`, expression B-tree на `payload ->> 'device'`. Для кожного є `EXPLAIN (ANALYZE, BUFFERS)` до і після, параметри planner не змінюються. Ціна запису для кожного набору індексів виміряна окремо.
- `t7_hw_checkouts_norm`: той самий агрегат у JSONB- і normalized-версії, порівняння планів і розмірів.
- MongoDB-еквівалент як pseudocode і Reflection наприкінці notebook.

## Запуск

Google Colab: відкрити `hw_t7.ipynb` і виконати Runtime > Run all. Перша клітинка встановить залежності. У Colab потрібна версія середовища 2026.07 (Python 3.12): Runtime → Change runtime type → Runtime version, бо `pgserver` не має збірок для Python 3.13.

Локально потрібен Python 3.12 (для новіших версій може не бути wheel пакета `pgserver`):

```bash
python3.12 -m venv .venv
source .venv/bin/activate
pip install pgserver psycopg2-binary "psycopg[binary]" sqlalchemy pandas pyarrow requests jupyter
jupyter notebook hw_t7.ipynb
```

PostgreSQL піднімається через `pgserver` у `/tmp/hw7_pg`, окремо встановлювати його не треба. Notebook можна запускати повторно: таблиці й view перестворюються. Часи в планах залежать від машини, тож під час нового запуску цифри в Markdown відрізнятимуться від виводу.

## Структура

```
hw_t7.ipynb   notebook з виводами клітинок
README.md
.gitignore
```
