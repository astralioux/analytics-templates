# 🗓️ SQL Dates & Time — Cheatsheet (Postgres • ClickHouse • BigQuery)

## 1) Типы

* `DATE` — дата
* `TIMESTAMP`/`DATETIME` — дата+время
* `TIMESTAMPTZ` — дата+время+часовой пояс (Postgres)
* `INTERVAL` — промежуток

## 2) Текущие значения

**Postgres**

```sql
CURRENT_DATE
NOW()                -- timestamp with time zone
LOCALTIMESTAMP       -- без TZ
```

**ClickHouse**

```sql
today()              -- Date
now()                -- DateTime
```

**BigQuery**

```sql
CURRENT_DATE()
CURRENT_TIMESTAMP()
```

## 3) Извлечение частей / округление

**Postgres**

```sql
EXTRACT(YEAR FROM ts)
DATE_TRUNC('day', ts)       -- начало дня
DATE_TRUNC('week', ts)      -- начало недели (по locale)
```

**ClickHouse**

```sql
toYear(ts), toMonth(ts), toDayOfMonth(ts), toDayOfWeek(ts)
toStartOfDay(ts)
toStartOfWeek(ts[, 'Europe/Moscow'])
```

**BigQuery**

```sql
EXTRACT(YEAR FROM ts)
DATE_TRUNC(ts, DAY)
DATE_TRUNC(ts, WEEK)        -- неделя с воскресенья; WEEK(MONDAY) — с понедельника
```

## 4) Арифметика

**Postgres**

```sql
ts + INTERVAL '7 days'
AGE(ts2, ts1)                    -- интервал
```

**ClickHouse**

```sql
ts + INTERVAL 7 DAY
dateDiff('day', d1, d2)          -- разница в днях
```

**BigQuery**

```sql
TIMESTAMP_ADD(ts, INTERVAL 7 DAY)
DATE_DIFF(d2, d1, DAY)
```

## 5) Парсинг / Форматирование

**Postgres**

```sql
TO_DATE('24-10-2025','DD-MM-YYYY')
TO_TIMESTAMP('24/10/2025 14:30','DD/MM/YYYY HH24:MI')
TO_CHAR(ts,'YYYY-MM-DD')
```

**ClickHouse**

```sql
parseDateTimeBestEffort('2025-10-24 14:30')
formatDateTime(ts, '%Y-%m-%d')
```

**BigQuery**

```sql
PARSE_DATE('%d-%m-%Y','24-10-2025')
PARSE_TIMESTAMP('%d/%m/%Y %H:%M','24/10/2025 14:30', 'Europe/Moscow')
FORMAT_TIMESTAMP('%Y-%m-%d', ts, 'Europe/Moscow')
```

## 6) Частые фильтры (последние N)

**Последние 7 дней (включая сегодня)**

```sql
-- Postgres
WHERE event_date >= CURRENT_DATE - INTERVAL '7 days'

-- ClickHouse (Date)
WHERE event_date >= today() - 7

-- ClickHouse (DateTime)
WHERE event_time >= now() - INTERVAL 7 DAY

-- BigQuery (DATE)
WHERE event_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
```

**Последний полный месяц**

```sql
-- Postgres
WHERE date_col >= DATE_TRUNC('month', CURRENT_DATE - INTERVAL '1 month')
  AND date_col <  DATE_TRUNC('month', CURRENT_DATE)

-- ClickHouse
WHERE date_col >= toStartOfMonth(addMonths(today(), -1))
  AND date_col <  toStartOfMonth(today())

-- BigQuery
WHERE date_col >= DATE_TRUNC(DATE_SUB(CURRENT_DATE(), INTERVAL 1 MONTH), MONTH)
  AND date_col <  DATE_TRUNC(CURRENT_DATE(), MONTH)
```

## 7) Группировки по дням/неделям

```sql
-- Postgres
SELECT DATE_TRUNC('day', ts) AS day, COUNT(*) FROM t GROUP BY 1;

-- ClickHouse
SELECT toStartOfDay(ts) AS day, COUNT(*) FROM t GROUP BY day;

-- BigQuery
SELECT DATE_TRUNC(ts, DAY) AS day, COUNT(*) FROM t GROUP BY day;
```

## 8) Часовые пояса

**Конвертация в локальный TZ**

```sql
-- Postgres (ts timestamptz → Europe/Moscow)
SELECT (ts AT TIME ZONE 'Europe/Moscow') AS local_ts;

-- ClickHouse
SELECT toTimeZone(ts, 'Europe/Moscow') AS local_ts;

-- BigQuery
SELECT DATETIME(ts, 'Europe/Moscow') AS local_dt;  -- представление без TZ
```

## 9) Безопасные границы интервалов

Всегда используйте **левая граница включительно, правая — исключительно**:

```sql
-- День
WHERE ts >= '2025-10-24'::timestamp
  AND ts <  '2025-10-25'::timestamp
```

Это спасает от проблем с миллисекундами и `BETWEEN`.

## 10) Неделя: с какого дня?

* Postgres: `DATE_TRUNC('week', ...)` зависит от локали/настроек.
* ClickHouse: используйте `toStartOfWeek(ts, 'Europe/Moscow')` и `toDayOfWeek()` (1=Пн, 7=Вс при ISO).
* BigQuery: `DATE_TRUNC(ts, WEEK(MONDAY))` для ISO-недели.

## 11) Частые кейсы (шаблоны)

**Когорты по дате регистрации (дни)**

```sql
-- Postgres
SELECT DATE_TRUNC('day', signup_ts) AS cohort_day, COUNT(DISTINCT user_id)
FROM users
GROUP BY 1;
```

**Разница между регистрацией и первой оплатой**

```sql
-- ClickHouse
SELECT user_id, dateDiff('day', signup_date, first_payment_date) AS days_to_pay
FROM users;
```

**Окно “скользящие 7 дней”**

```sql
-- BigQuery (по датам)
SELECT date_col,
       SUM(metric) OVER (ORDER BY date_col
                         ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS last_7_days
FROM daily_metrics;
```

## 12) Мини-чеклист

* ✅ Даты храним в UTC, отображаем в локальном TZ
* ✅ Границы интервалов: `>= start AND < end`
* ✅ Для агрегаций используем `DATE_TRUNC` / `toStartOf*`
* ✅ Явно указываем TZ при форматировании/парсинге
* ✅ Для “последнего полного периода” сначала вычисляем его начало/конец

---
