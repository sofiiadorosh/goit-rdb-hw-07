# goit-rdb-hw-07

Домашнє завдання 5 (Тема 7): часові ряди та напівструктуровані дані у задачах ML.

Усе рішення в одному notebook: [hw7_taxi_timeseries_jsonb.ipynb](hw7_taxi_timeseries_jsonb.ipynb). Output-и всіх клітинок, зокрема `EXPLAIN (ANALYZE, BUFFERS)`, збережені.

## Що всередині

| Розділ | Зміст |
|---|---|
| Завдання 1 | `pgserver`, завантаження NYC TLC Yellow Taxi (лютий + березень 2024), daily-агрегат `t7_hw_taxi_daily`, typed-таблиця з `TIMESTAMPTZ` |
| Завдання 2 | Lag (1d / 7d / 30d), rolling 7d avg, rolling 30d std, expanding-ознаки; view `v_t7_hw_taxi_features` |
| Завдання 3 | JSONB event-log `t7_hw_events` (50 000 подій, 5 типів); запити з `@>`, `?`, `?\|` і JSONPath |
| Завдання 4 | GIN (`jsonb_path_ops` / `jsonb_ops`) і expression B-tree index, EXPLAIN до та після, ціна запису |
| Завдання 5 | Нормалізована таблиця `t7_hw_checkouts_norm`, порівняння EXPLAIN, MongoDB pseudocode, Reflection |

## Як запустити

### Google Colab

Відкрийте `hw7_taxi_timeseries_jsonb.ipynb` і виконайте **Runtime → Restart and run all**. Перша клітинка сама встановить залежності, а parquet-файли завантажаться в `/content`. Якщо `pgserver` у Colab не встановлюється, notebook автоматично піднімає системний PostgreSQL через `apt-get`.

### Локально

`pgserver` не має wheel-ів для Python 3.14, тому потрібен Python 3.10–3.12.

```bash
python3.12 -m venv .venv
```

```bash
.venv/bin/pip install pgserver psycopg2-binary sqlalchemy pandas pyarrow requests jupyter
```

```bash
.venv/bin/jupyter nbconvert --to notebook --execute --inplace hw7_taxi_timeseries_jsonb.ipynb
```

Щоб підключитися до вже запущеного PostgreSQL замість `pgserver`, задайте `HW7_DB_URI` (наприклад, `postgresql://postgres:postgres@127.0.0.1:5432/postgres`).

Parquet-файли (~120 MB) завантажуються в `data/`, ця папка в `.gitignore`. Якщо мережі немає, notebook автоматично переходить на stub-схему (60 днів × 5 зон).

## Відхилення від шаблону завдання

- **Два місяці та вибірка замість `head(500_000)`.** Перші 500 000 рядків березневого файлу покривають лише 1–6 березня, тож `trips_lag_30d` завжди був би `NULL`. Тому з кожного місяця береться відтворювана вибірка `sample(250_000, random_state=42)`.
- **Calendar spine.** `t7_hw_taxi_daily` містить рядок на кожну пару «зона × день» (240 × 60 = 14 400). Дні без поїздок мають `trip_count = 0`, тож `LAG(n)` дорівнює рівно `n` календарним дням.
- **Ідемпотентність.** `pgserver` зберігає базу в `/tmp/hw7_pg` між запусками. Тому всі об'єкти перестворюються через `DROP ... IF EXISTS`, а view видаляється до своєї базової таблиці.
- **Підключення.** Драйвер `psycopg2` вказано явно, а сесійна таймзона — `UTC`, щоб `TIMESTAMPTZ` в output-ах не залежав від машини.

## Ключові результати

| Запит | До індексу | Після індексу |
|---|---|---|
| `payload @> {"payment_method": "card"}` | Seq Scan, 12.5 ms | Bitmap Index Scan (GIN), 1.6 ms |
| `payload ->> 'device' = 'mobile'` | Seq Scan, 2.7 ms | Bitmap Index Scan (B-tree), 1.3 ms |

| Агрегат по checkout | Сторінок прочитано | Час |
|---|---|---|
| JSONB (`t7_hw_events`) | 1000 | 3.6 ms |
| Normalized (`t7_hw_checkouts_norm`) | 94 | 1.0 ms |

Цифри взято із запуску, збереженого в notebook. Детальний розбір — у розділі Reflection.
