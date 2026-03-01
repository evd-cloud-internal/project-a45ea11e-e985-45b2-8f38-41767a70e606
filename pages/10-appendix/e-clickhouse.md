---
name: E Clickhouse
assetId: f24c3951-167f-4cf4-83e1-985d34c69015
type: page
---

---
title: ClickHouse SQL Cheat Sheet
---

# ClickHouse SQL Cheat Sheet

A survival guide for teams coming from DuckDB / MotherDuck. ClickHouse is fast but strict — these are the patterns that bite most often and the fixes that work every time.

_If a query works in DuckDB but fails in ClickHouse, start here._

## 0. What DuckDB Syntax Accidentally Works (and What Doesn't)

There is **no DuckDB compatibility shim** in Evidence Studio's ClickHouse layer. You're writing raw ClickHouse SQL. However, ClickHouse natively supports aliases for some common SQL functions, which creates a false sense of safety — some DuckDB habits work by coincidence, others fail hard.

### ✅ DuckDB syntax that happens to work

| DuckDB syntax | Why it works in ClickHouse |
|---|---|
| `DATE_TRUNC('month', d)` | ClickHouse added this as a native alias |
| `DATEDIFF('day', a, b)` | Function names are case-insensitive in CH |
| `REPLACE('foo', 'a', 'b')` | CH alias for `replaceAll` — same behavior |
| `TRIM('  hello  ')` | Standard SQL, supported natively |
| `<>` | Standard SQL, works fine |
| `COALESCE(a, b)` | Standard SQL, works fine |
| `SUBSTRING(s, 1, 3)` | Standard SQL, works fine |
| `LENGTH(s)` | Case-insensitive match to `length()` |

### ❌ DuckDB syntax that fails

| DuckDB syntax | ClickHouse equivalent | Error you'll see |
|---|---|---|
| `CURRENT_DATE` | `today()` | Unknown expression identifier |
| `CURRENT_TIMESTAMP` | `now()` | Unknown expression identifier |
| `STRFTIME(d, '%Y-%m')` | `formatDateTime(d, '%Y-%m')` | Function does not exist |
| `STRING_SPLIT(s, ',')` | `splitByChar(',', s)` | Function does not exist (also: arg order is reversed!) |
| `LIST(x)` | `groupArray(x)` | Function does not exist |
| `SUM(x) FILTER (WHERE ...)` | `sumIf(x, ...)` | Syntax error |
| `d + INTERVAL '30 days'` | `d + INTERVAL 30 DAY` | No quotes, singular unit |
| `CAST(x AS DOUBLE)` | `CAST(x AS Float64)` | Unknown type |
| `EXTRACT(YEAR FROM d)` | `toYear(d)` | Works but `toYear()` is idiomatic |

**The rule:** If it's standard SQL (TRIM, COALESCE, SUBSTRING), it probably works. If it's a DuckDB-specific function name (STRFTIME, STRING_SPLIT, LIST) or keyword (CURRENT_DATE), it won't. When in doubt, use the ClickHouse-native function.

## 1. Type Strictness

ClickHouse refuses to guess types. DuckDB auto-coerces; ClickHouse throws an error.

### UNION ALL requires matching types

```sql union_type_bad
-- ❌ Fails: "no supertype for Int64, Float64"
SELECT 42 AS val
UNION ALL
SELECT 3.14 AS val
```

```sql union_type_good
-- ✅ Fix: explicit cast
SELECT CAST(42 AS Float64) AS val
UNION ALL
SELECT 3.14 AS val
```

### GREATEST / LEAST with mixed types

```sql greatest_bad
-- ❌ Fails: Int literal 0 vs Float64 expression
SELECT GREATEST(0, ROUND(3.7 - 1.5, 2)) AS floor_val
```

```sql greatest_good
-- ✅ Fix: cast the literal to match
SELECT GREATEST(CAST(0 AS Float64), ROUND(3.7 - 1.5, 2)) AS floor_val
```

**The rule:** If two expressions meet (UNION ALL, GREATEST, LEAST, CASE WHEN, IF), their types must match exactly. When in doubt, `CAST(x AS Float64)`.

## 2. Aggregate Alias Traps

ClickHouse is strict about where aggregate aliases are visible and whether expressions look like nested aggregates.

### ORDER BY with aggregate aliases

```sql order_by_bad
-- ❌ Fails: "aggregate inside another aggregate"
SELECT category, SUM(amount) AS total
FROM (SELECT 'A' AS category, 100 AS amount)
GROUP BY category
ORDER BY SUM(amount) DESC
```

```sql order_by_good
-- ✅ Fix: reference the alias, don't re-wrap in SUM()
SELECT category, SUM(amount) AS total
FROM (SELECT 'A' AS category, 100 AS amount)
GROUP BY category
ORDER BY total DESC
```

### Weighted averages trigger nested aggregate detection

```sql weighted_bad
-- ❌ Fails: ClickHouse sees SUM(count) inside the denominator
--    as a nested aggregate when count is also a column name
-- SELECT SUM(count * rate) / NULLIF(SUM(count), 0) AS weighted_rate
-- FROM my_table GROUP BY category
SELECT 'see comment above — this pattern fails' AS note
```

```sql weighted_good
-- ✅ Fix: aggregate in a subquery, compute ratios in the outer query
SELECT weighted_numerator / NULLIF(total_count, 0) AS weighted_rate
FROM (
    SELECT
        SUM(count * rate) AS weighted_numerator
      , SUM(count) AS total_count
    FROM (
        SELECT 10 AS count, 0.05 AS rate
        UNION ALL
        SELECT 20, 0.12
    )
)
```

**The rule:** If ClickHouse complains about "aggregate inside aggregate," wrap the inner aggregation in a subquery and compute derived values in the outer SELECT.

## 3. Date & Time Functions

ClickHouse uses camelCase function names and different syntax for intervals. This is probably the biggest muscle-memory adjustment.

```sql date_comparison
-- Side-by-side: DuckDB → ClickHouse
SELECT
    -- Truncate to month
    -- DuckDB:      DATE_TRUNC('month', d)
    toStartOfMonth(toDate('2024-06-15'))          AS trunc_month

    -- Add interval
    -- DuckDB:      d + INTERVAL '30 days'
  , toDate('2024-06-15') + INTERVAL 30 DAY        AS plus_30

    -- Date difference
    -- DuckDB:      DATEDIFF('day', start, end)
  , dateDiff('day', toDate('2024-01-01'), toDate('2024-06-15')) AS day_diff

    -- Current date/time
    -- DuckDB:      CURRENT_DATE / CURRENT_TIMESTAMP
  , today()                                        AS current_date_ch
  , now()                                          AS current_ts_ch

    -- Format a date
    -- DuckDB:      STRFTIME(d, '%Y-%m')
  , formatDateTime(toDate('2024-06-15'), '%Y-%m')  AS formatted

    -- Extract parts
    -- DuckDB:      EXTRACT(YEAR FROM d)
  , toYear(toDate('2024-06-15'))                   AS yr
  , toMonth(toDate('2024-06-15'))                  AS mo
```

### Always use explicit date constructors

```sql date_explicit
-- ❌ Risky: implicit string-to-date comparison
-- WHERE date_col > '2024-01-01'

-- ✅ Safe: explicit conversion
SELECT toDate('2024-01-01') AS safe_date, toDateTime('2024-01-15 10:30:00') AS safe_ts
```

### Common date truncation functions

| DuckDB | ClickHouse |
|---|---|
| `DATE_TRUNC('year', d)` | `toStartOfYear(d)` |
| `DATE_TRUNC('quarter', d)` | `toStartOfQuarter(d)` |
| `DATE_TRUNC('month', d)` | `toStartOfMonth(d)` |
| `DATE_TRUNC('week', d)` | `toStartOfWeek(d)` or `toMonday(d)` |
| `DATE_TRUNC('day', d)` | `toDate(d)` |

### Interval syntax

```sql interval_syntax
-- ClickHouse: no quotes around the unit, singular form
SELECT
    today() + INTERVAL 1 DAY       AS tomorrow
  , today() - INTERVAL 3 MONTH     AS three_months_ago
  , today() + INTERVAL 1 YEAR      AS next_year
```

## 4. String Handling

ClickHouse strings are **case-sensitive by default**. DuckDB is often lenient.

```sql string_examples
SELECT
    -- Case-sensitive comparison (default)
    'Active' = 'active'                     AS is_equal        -- 0 (false!)

    -- Case-insensitive alternatives
  , lower('Active') = 'active'              AS lower_match     -- 1
  , 'Active' ILIKE 'active'                 AS ilike_match     -- 1

    -- String functions (camelCase!)
  , length('hello')                         AS len             -- 5
  , substring('hello', 1, 3)                AS sub             -- hel
  , concat('hello', ' ', 'world')           AS joined          -- hello world
  , replaceAll('foo-bar-baz', '-', '_')     AS replaced        -- foo_bar_baz
  , splitByChar('-', 'foo-bar-baz')         AS split_arr       -- ['foo','bar','baz']
  , trimBoth('  hello  ')                   AS trimmed         -- hello
```

### Use != not <>

Both technically work, but `!=` is idiomatic ClickHouse and avoids edge-case parser issues.

## 5. GROUP BY Strictness

ClickHouse requires every non-aggregated column in SELECT to appear in GROUP BY. DuckDB is sometimes lenient.

```sql group_by_strict
-- ❌ Fails: column 'name' not in GROUP BY
-- SELECT category, name, SUM(amount) AS total
-- FROM my_table
-- GROUP BY category

-- ✅ Fix: include all non-aggregated columns
-- SELECT category, name, SUM(amount) AS total
-- FROM my_table
-- GROUP BY category, name

-- ✅ Or use any() to pick an arbitrary value
SELECT
    category
  , any(name) AS sample_name
  , SUM(amount) AS total
FROM (
    SELECT 'A' AS category, 'Alice' AS name, 100 AS amount
    UNION ALL
    SELECT 'A', 'Bob', 200
)
GROUP BY category
```

## 6. Boolean Handling

ClickHouse booleans are `UInt8` (0 and 1), not true boolean types.

```sql bool_examples
SELECT
    -- These all work
    1 AS flag_true
  , 0 AS flag_false
  , IF(1, 'yes', 'no') AS conditional

    -- Casting
  , CAST(1 AS Bool) AS explicit_bool
  , toBool(0) AS also_works
```

## 7. NULL Behavior

Mostly the same as DuckDB, but a few surprises.

```sql null_examples
SELECT
    -- Standard NULL checks
    NULL IS NULL                              AS is_null       -- 1
  , COALESCE(NULL, NULL, 42)                 AS coalesced     -- 42
  , NULLIF(0, 0)                             AS nullified     -- NULL
  , ifNull(NULL, 'fallback')                 AS if_null       -- fallback

    -- ClickHouse-specific: assumeNotNull (dangerous but fast)
    -- Treats NULL as default value for the type (0 for numbers, '' for strings)
  , assumeNotNull(CAST(NULL AS Nullable(Int64))) AS assumed   -- 0
```

### NULLS LAST in ORDER BY

```sql nulls_last
-- ClickHouse puts NULLs first by default (opposite of some databases)
-- Always specify NULLS LAST when sorting DESC
SELECT val
FROM (SELECT 100 AS val UNION ALL SELECT NULL AS val UNION ALL SELECT 200 AS val)
ORDER BY val DESC NULLS LAST
```

## 8. Useful ClickHouse-Specific Functions

Things ClickHouse has that DuckDB doesn't (or does differently).

```sql ch_specific
SELECT
    -- Conditional aggregation (cleaner than CASE)
    sumIf(amount, category = 'A')           AS sum_a
  , countIf(category = 'B')                 AS count_b
  , avgIf(amount, amount > 0)               AS avg_positive

    -- Array functions
  , arrayJoin([1, 2, 3])                    AS expanded       -- 3 rows
  , groupArray(category)                    AS all_cats        -- ['A', 'B']

    -- Rounding
  , round(3.14159, 2)                       AS rounded         -- 3.14
  , ceil(3.1)                               AS ceiling         -- 4
  , floor(3.9)                              AS floored         -- 3
FROM (
    SELECT 'A' AS category, 100 AS amount
    UNION ALL
    SELECT 'B', 200
)
```

## 9. Quick Reference: DuckDB → ClickHouse

| Concept | DuckDB | ClickHouse |
|---|---|---|
| Current date | `CURRENT_DATE` | `today()` |
| Current timestamp | `CURRENT_TIMESTAMP` | `now()` |
| Date truncate | `DATE_TRUNC('month', d)` | `toStartOfMonth(d)` |
| Date difference | `DATEDIFF('day', a, b)` | `dateDiff('day', a, b)` |
| Date format | `STRFTIME(d, '%Y-%m')` | `formatDateTime(d, '%Y-%m')` |
| Add interval | `d + INTERVAL '30 days'` | `d + INTERVAL 30 DAY` |
| String length | `LENGTH(s)` | `length(s)` |
| Substring | `SUBSTRING(s, 1, 3)` | `substring(s, 1, 3)` |
| Replace | `REPLACE(s, 'a', 'b')` | `replaceAll(s, 'a', 'b')` |
| Split | `STRING_SPLIT(s, ',')` | `splitByChar(',', s)` |
| Trim | `TRIM(s)` | `trimBoth(s)` |
| Conditional agg | `SUM(x) FILTER (WHERE ...)` | `sumIf(x, ...)` |
| List agg | `LIST(x)` | `groupArray(x)` |
| Not equals | `<>` or `!=` | `!=` |
| Boolean true | `TRUE` | `1` or `true` |
| Type cast | `CAST(x AS DOUBLE)` | `CAST(x AS Float64)` |
| Auto-increment | `ROWID` | `rowNumberInAllBlocks()` |

## 10. Common ClickHouse Types

| Type | Notes |
|---|---|
| `Int8`, `Int16`, `Int32`, `Int64` | Signed integers |
| `UInt8`, `UInt16`, `UInt32`, `UInt64` | Unsigned integers (UInt8 = boolean) |
| `Float32`, `Float64` | Floating point (Float64 ≈ DuckDB's DOUBLE) |
| `Decimal(P, S)` | Fixed precision (e.g., `Decimal(18, 2)` for currency) |
| `String` | Variable-length (no VARCHAR/TEXT distinction) |
| `Date` | Calendar date |
| `DateTime` | Date + time (second precision) |
| `DateTime64(3)` | Millisecond precision timestamps |
| `Nullable(T)` | Wraps any type to allow NULL |
| `Array(T)` | Array of any type |