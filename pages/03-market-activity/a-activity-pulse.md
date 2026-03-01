---
name: A Activity Pulse
assetId: 9f9ffa20-318a-4ead-95fb-c9ef4cb28324
type: page
---

---
title: Activity Pulse
---

# Who Will Transact Next?

Transaction Propensity Intelligence is the largest signal family in the platform, covering 93 instrument types across 7 subcategories. It tracks deeds, releases, foreclosures, life events, and deal signals that predict which properties are likely to sell, refinance, or experience a forced transfer. This is the category a lending partner opens first: it answers where the next transaction is coming from and why.

## Data Health

```sql data_health
-- Data health strip: amount coverage for Transaction Propensity Intelligence
-- Source: audit_amount_data_quality (synced from audit.amount_data_quality)
-- Filter: category = 'Transaction Propensity Intelligence'
SELECT
    SUM(total_count) AS total_instruments
  , SUM(has_any_amount_count) AS with_amount
  , ROUND(SUM(has_any_amount_count) * 100.0 / SUM(total_count), 1) AS pct_with_amount
FROM audit_amount_data_quality
WHERE category = 'Transaction Propensity Intelligence'
```

{% row %}
    {% big_value
        data="data_health"
        value="total_instruments"
        title="Classified Instruments"
        fmt="num0k"
    /%}
    {% big_value
        data="data_health"
        value="pct_with_amount"
        title="% With Dollar Amount"
        fmt="num0"
    /%}
{% /row %}

> These metrics cover all 93 instrument types classified under Transaction Propensity Intelligence. Dollar amount coverage varies by subcategory: deed transfers typically carry recorded amounts, while notices and filings often do not.

## How Is Overall Market Activity Behaving?

Two control charts track the health of this signal family. Volume measures how many instruments are being recorded. Dollar value measures the capital moving through those instruments. When the two charts tell different stories (volume flat but dollars spiking), it usually means a few large transactions are driving the numbers.

Control limits calculated using XmR method (see Control Chart Method in appendix).

### Volume

```sql volume_series
-- Volume XmR chart series: monthly values + 6-month moving average
-- Source: gold_signal_velocity_trends
-- Filter: category = 'Transaction Propensity Intelligence'
WITH monthly AS (
    SELECT
        record_month
      , SUM(current_month_count) AS volume
    FROM gold_signal_velocity_trends
    WHERE category = 'Transaction Propensity Intelligence'
    GROUP BY record_month
)
, with_mr AS (
    SELECT
        record_month
      , volume
      , ABS(volume - LAG(volume) OVER (ORDER BY record_month)) AS moving_range
    FROM monthly
)
, stats AS (
    SELECT
        AVG(volume) AS mean_vol
      , AVG(moving_range) AS avg_mr
    FROM with_mr
)
, control AS (
    SELECT
        with_mr.record_month
      , toFloat64(with_mr.volume) AS volume
      , ROUND(AVG(with_mr.volume) OVER (
          ORDER BY with_mr.record_month
          ROWS BETWEEN 5 PRECEDING AND CURRENT ROW), 0) AS volume_6mo_avg
    FROM with_mr
    CROSS JOIN stats
    WHERE with_mr.record_month > (SELECT MIN(record_month) FROM monthly)
)
SELECT record_month, 'Monthly' AS series, volume AS value FROM control
UNION ALL
SELECT record_month, '6-Mo Average' AS series, volume_6mo_avg AS value FROM control
ORDER BY record_month, series
```

{% line_chart
    data="volume_series"
    x="record_month"
    y="value"
    series="series"
    y_fmt="num0"
    title="Monthly Transaction Propensity Volume"
/%}

### Dollar Value

```sql dollar_series
-- Dollar XmR chart series: monthly values + 6-month moving average
-- Source: gold_signal_velocity_trends
-- Filter: category = 'Transaction Propensity Intelligence'
WITH monthly AS (
    SELECT
        record_month
      , SUM(current_month_amount) AS dollars
    FROM gold_signal_velocity_trends
    WHERE category = 'Transaction Propensity Intelligence'
    GROUP BY record_month
)
, with_mr AS (
    SELECT
        record_month
      , dollars
      , ABS(dollars - LAG(dollars) OVER (ORDER BY record_month)) AS moving_range
    FROM monthly
)
, stats AS (
    SELECT
        AVG(dollars) AS mean_dol
      , AVG(moving_range) AS avg_mr
    FROM with_mr
)
, control AS (
    SELECT
        with_mr.record_month
      , with_mr.dollars
      , ROUND(AVG(with_mr.dollars) OVER (
          ORDER BY with_mr.record_month
          ROWS BETWEEN 5 PRECEDING AND CURRENT ROW), 0) AS dollars_6mo_avg
    FROM with_mr
    CROSS JOIN stats
    WHERE with_mr.record_month > (SELECT MIN(record_month) FROM monthly)
)
SELECT record_month, 'Monthly' AS series, dollars AS value FROM control
UNION ALL
SELECT record_month, '6-Mo Average' AS series, dollars_6mo_avg AS value FROM control
ORDER BY record_month, series
```

{% line_chart
    data="dollar_series"
    x="record_month"
    y="value"
    series="series"
    y_fmt="usd0m"
    title="Monthly Transaction Propensity Dollar Volume"
/%}

## Attention Now

```sql attention_now
-- Attention Now: subcategories with most recent complete month outside XmR control limits
-- Source: gold_signal_velocity_trends
-- Filter: category = 'Transaction Propensity Intelligence'
-- Evaluates offset 1 (most recent complete month, skipping current partial)
WITH monthly AS (
    SELECT
        record_month
      , subcategory
      , SUM(current_month_count) AS volume
    FROM gold_signal_velocity_trends
    WHERE category = 'Transaction Propensity Intelligence'
    GROUP BY record_month, subcategory
)
, eval_month AS (
    -- Most recent complete month (offset 1 skips current partial)
    SELECT record_month
    FROM monthly
    ORDER BY record_month DESC
    LIMIT 1 OFFSET 1
)
, with_mr AS (
    SELECT
        record_month
      , subcategory
      , volume
      , ABS(volume - LAG(volume) OVER (
            PARTITION BY subcategory ORDER BY record_month)) AS moving_range
    FROM monthly
)
, stats AS (
    SELECT
        subcategory
      , AVG(volume) AS mean_vol
      , AVG(moving_range) AS avg_mr
    FROM with_mr
    GROUP BY subcategory
)
, limits AS (
    SELECT
        subcategory
      , ROUND(mean_vol, 0) AS mean_vol
      , ROUND(mean_vol + 3 * (avg_mr / 1.128), 0) AS ucl
      , GREATEST(toFloat64(0), ROUND(mean_vol - 3 * (avg_mr / 1.128), 0)) AS lcl
    FROM stats
)
, eval_values AS (
    SELECT
        w.subcategory AS subcategory
      , w.volume AS eval_volume
      , l.mean_vol AS mean_vol
      , l.ucl AS ucl
      , l.lcl AS lcl
      , CASE
            WHEN w.volume > l.ucl THEN 'Above UCL'
            WHEN w.volume < l.lcl THEN 'Below LCL'
            ELSE NULL
        END AS signal_status
    FROM with_mr w
    INNER JOIN eval_month e ON w.record_month = e.record_month
    INNER JOIN limits l ON w.subcategory = l.subcategory
)
SELECT
    ev.subcategory AS subcategory
  , CASE ev.subcategory
        WHEN 'Equity Unlock Signals' THEN 'Releases, satisfactions, reconveyances'
        WHEN 'Owner Pressure Signals' THEN 'Tax liens, delinquencies, lis pendens'
        WHEN 'Forced Sale Signals' THEN 'Foreclosure deeds, tax deeds, auctions'
        WHEN 'Blocker Clearance Signals' THEN 'Cancellations, terminations, withdrawals'
        WHEN 'Transaction Timing Triggers' THEN 'Deeds, assignments, sales'
        WHEN 'Life and Legal Triggers' THEN 'Probate, death certificates, divorce'
        WHEN 'Deal Propensity Events' THEN 'Options, purchase contracts'
        ELSE ev.subcategory
    END AS what_it_tracks
  , ev.eval_volume AS volume
  , ev.mean_vol AS mean
  , ev.ucl AS upper_limit
  , ev.lcl AS lower_limit
  , ev.signal_status AS status
FROM eval_values ev
WHERE ev.signal_status IS NOT NULL
ORDER BY ev.eval_volume DESC NULLS LAST
```

{% if data="attention_now" %}

Subcategories flagged here had volume outside their normal statistical range in the most recent complete month. A value above the upper control limit (UCL) means the subcategory recorded more instruments than expected. Below the lower control limit (LCL) means fewer. Either warrants investigation: the subcategory pages and Signal Detail drill-through show which specific instruments drove the spike or drop.

{% table data="attention_now" %}
    {% dimension value="subcategory" title="Subcategory" /%}
    {% dimension value="what_it_tracks" title="Tracks" /%}
    {% measure value="volume" title="Volume" fmt="num0" /%}
    {% measure value="mean" title="Mean" fmt="num0" /%}
    {% measure value="upper_limit" title="UCL" fmt="num0" /%}
    {% measure value="status" title="Status" /%}
{% /table %}

{% /if %}
{% else %}

All seven subcategories are within normal statistical variation this month. No anomalies to investigate.

{% /else %}

## This Month at a Glance

```sql this_month_glance
-- This Month at a Glance: latest two months for comparison
-- Source: gold_signal_velocity_trends
-- Filter: category = 'Transaction Propensity Intelligence'
WITH months AS (
    SELECT DISTINCT record_month
    FROM gold_signal_velocity_trends
    WHERE category = 'Transaction Propensity Intelligence'
    ORDER BY record_month DESC NULLS LAST
    LIMIT 2
)
SELECT
    SUM(CASE WHEN record_month = (SELECT MAX(record_month) FROM months) THEN current_month_count ELSE 0 END) AS current_instruments
  , SUM(CASE WHEN record_month = (SELECT MIN(record_month) FROM months) THEN current_month_count ELSE 0 END) AS prior_instruments
  , SUM(CASE WHEN record_month = (SELECT MAX(record_month) FROM months) THEN current_month_amount ELSE 0 END) AS current_dollars
  , SUM(CASE WHEN record_month = (SELECT MIN(record_month) FROM months) THEN current_month_amount ELSE 0 END) AS prior_dollars
FROM gold_signal_velocity_trends
WHERE category = 'Transaction Propensity Intelligence'
  AND record_month IN (SELECT record_month FROM months)
```

{% row %}
    {% big_value
        data="this_month_glance"
        value="current_instruments"
        title="Instruments"
        fmt="num0"
        comparison={
            compare_vs="target"
            target="prior_instruments"
            pct_fmt="pct0"
        }
    /%}
    {% big_value
        data="this_month_glance"
        value="current_dollars"
        title="Dollar Volume"
        fmt="usd0m"
        comparison={
            compare_vs="target"
            target="prior_dollars"
            pct_fmt="pct0"
        }
    /%}
{% /row %}

## Which Subcategories Are Moving?

```sql mom_subcategory
-- Month-over-month by subcategory: current vs prior month, sorted by absolute change
-- Source: gold_signal_velocity_trends
-- Filter: category = 'Transaction Propensity Intelligence'
WITH latest_two AS (
    SELECT DISTINCT record_month
    FROM gold_signal_velocity_trends
    WHERE category = 'Transaction Propensity Intelligence'
    ORDER BY record_month DESC NULLS LAST
    LIMIT 2
)
, current_month AS (
    SELECT record_month FROM latest_two ORDER BY record_month DESC NULLS LAST LIMIT 1
)
, prior_month AS (
    SELECT record_month FROM latest_two ORDER BY record_month LIMIT 1
)
SELECT
    gold_signal_velocity_trends.subcategory
  , CASE gold_signal_velocity_trends.subcategory
        WHEN 'Equity Unlock Signals' THEN 'Releases, satisfactions, reconveyances'
        WHEN 'Owner Pressure Signals' THEN 'Tax liens, delinquencies, lis pendens'
        WHEN 'Forced Sale Signals' THEN 'Foreclosure deeds, tax deeds, auctions'
        WHEN 'Blocker Clearance Signals' THEN 'Cancellations, terminations, withdrawals'
        WHEN 'Transaction Timing Triggers' THEN 'Deeds, assignments, sales'
        WHEN 'Life and Legal Triggers' THEN 'Probate, death certificates, divorce'
        WHEN 'Deal Propensity Events' THEN 'Options, purchase contracts'
        ELSE gold_signal_velocity_trends.subcategory
    END AS what_it_tracks
  , SUM(gold_signal_velocity_trends.current_month_count) AS current_count
  , SUM(gold_signal_velocity_trends.prior_month_count) AS prior_count
  , SUM(gold_signal_velocity_trends.current_month_count)
    - SUM(gold_signal_velocity_trends.prior_month_count) AS absolute_change
  , SUM(gold_signal_velocity_trends.current_month_amount) AS current_amount
  , SUM(gold_signal_velocity_trends.prior_month_amount) AS prior_amount
FROM gold_signal_velocity_trends
WHERE gold_signal_velocity_trends.category = 'Transaction Propensity Intelligence'
  AND gold_signal_velocity_trends.record_month IN (SELECT record_month FROM current_month)
GROUP BY gold_signal_velocity_trends.subcategory
ORDER BY ABS(absolute_change) DESC NULLS LAST
```

Subcategories sorted by largest absolute change from the prior month. The top movers are the ones most likely to contain actionable intelligence this month.

{% table data="mom_subcategory" %}
    {% dimension value="subcategory" title="Subcategory" /%}
    {% dimension value="what_it_tracks" title="Tracks" /%}
    {% measure value="current_count" title="Current" fmt="num0" /%}
    {% measure value="prior_count" title="Prior" fmt="num0" /%}
    {% measure value="absolute_change" title="Change" fmt="num0" /%}
    {% measure value="current_amount" title="$ Current" fmt="usd0" /%}
{% /table %}

## Are Pressure Signals Diverging from Equity Unlocks?

```sql divergence_equity_pressure
-- Divergence Monitor: Equity Unlock Signals vs Owner Pressure Signals
-- Source: gold_signal_velocity_trends
-- Filter: subcategory IN ('Equity Unlock Signals', 'Owner Pressure Signals')
-- When these two lines diverge, the market health story is changing.
-- Rising equity unlocks + falling pressure = healthy. Inverse = stress building.
SELECT
    record_month
  , subcategory AS series
  , SUM(current_month_count) AS volume
FROM gold_signal_velocity_trends
WHERE subcategory IN ('Equity Unlock Signals', 'Owner Pressure Signals')
GROUP BY record_month, subcategory
ORDER BY record_month
```

When equity unlocks (releases, satisfactions) rise while owner pressure signals (tax liens, delinquencies) fall, the market is healthy: borrowers are clearing debt and freeing collateral. When the lines cross or converge, stress may be building. On Guam, pressure spikes have historically correlated with government tax enforcement cycles rather than interest rate movements.

{% line_chart
    data="divergence_equity_pressure"
    x="record_month"
    y="volume"
    series="series"
    y_fmt="num0"
    title="Equity Unlocks vs Owner Pressure"
    chart_options={
        series_colors={
            "Equity Unlock Signals"="#2CA02C"
            "Owner Pressure Signals"="#FF7F0E"
        }
    }
/%}

## What Follows What?

### Top Transitions (Sankey)
```sql transition_sankey_top
-- Sankey: top 15 Transaction Propensity transitions (property linkage)
-- Source: silver_instrument_transitions_combined
-- Excludes self-referential and circular flows which break Sankey rendering
-- For bidirectional pairs (A→B and B→A), keeps only the higher-count direction
WITH base AS (
    SELECT
        source_display_name AS source
      , target_display_name AS target
      , transition_count
    FROM silver_instrument_transitions_combined
    WHERE source_category = 'Transaction Propensity Intelligence'
      AND linkage_type = 'property'
      AND source_display_name != target_display_name
)
, deduped AS (
    SELECT
        a.source
      , a.target
      , a.transition_count
    FROM base a
    LEFT JOIN base b
        ON a.source = b.target AND a.target = b.source
    WHERE b.source IS NULL
       OR a.transition_count > b.transition_count
       OR (a.transition_count = b.transition_count AND a.source < a.target)
)
SELECT source, target, transition_count
FROM deduped
ORDER BY transition_count DESC NULLS LAST
LIMIT 15
```

{% sankey_chart
    data="transition_sankey_top"
    source="source"
    target="target"
    value="sum(transition_count)"
    value_fmt="num0"
    title="Top 15 Transitions from Transaction Propensity Instruments"
    node_labels="full"
/%}

### Expanded Transitions (Top 25)
```sql transition_sankey_all
-- Sankey: top 25 Transaction Propensity transitions (property linkage)
-- Source: silver_instrument_transitions_combined
-- Excludes self-referential and circular flows which break Sankey rendering
-- For bidirectional pairs (A→B and B→A), keeps only the higher-count direction
-- Capped at 25: indirect cycles (A→B→C→A) appear at 30+ and break Sankey rendering
WITH base AS (
    SELECT
        source_display_name AS source
      , target_display_name AS target
      , transition_count
    FROM silver_instrument_transitions_combined
    WHERE source_category = 'Transaction Propensity Intelligence'
      AND linkage_type = 'property'
      AND source_display_name != target_display_name
)
, deduped AS (
    SELECT
        a.source
      , a.target
      , a.transition_count
    FROM base a
    LEFT JOIN base b
        ON a.source = b.target AND a.target = b.source
    WHERE b.source IS NULL
       OR a.transition_count > b.transition_count
       OR (a.transition_count = b.transition_count AND a.source < a.target)
)
SELECT source, target, transition_count
FROM deduped
ORDER BY transition_count DESC NULLS LAST
LIMIT 25
```

{% sankey_chart
    data="transition_sankey_all"
    source="source"
    target="target"
    value="sum(transition_count)"
    value_fmt="num0"
    title="Top 25 Transitions from Transaction Propensity Instruments"
    node_labels="full"
/%}

```sql transition_preview
-- Transition preview: top 15 instrument-to-instrument transitions
-- Source: silver_instrument_transitions_combined
-- Filter: source_category = 'Transaction Propensity Intelligence', property linkage only
-- Shows what instruments typically follow Transaction Propensity instruments on the same property
SELECT
    silver_instrument_transitions_combined.source_subcategory
  , silver_instrument_transitions_combined.source_display_name
  , silver_instrument_transitions_combined.target_subcategory
  , silver_instrument_transitions_combined.target_display_name
  , silver_instrument_transitions_combined.transition_count
  , silver_instrument_transitions_combined.median_days_between
FROM silver_instrument_transitions_combined
WHERE silver_instrument_transitions_combined.source_category = 'Transaction Propensity Intelligence'
  AND silver_instrument_transitions_combined.linkage_type = 'property'
ORDER BY silver_instrument_transitions_combined.transition_count DESC NULLS LAST
LIMIT 15
```

When a Transaction Propensity instrument is recorded on a property, what comes next? This table shows the most common follow-on instruments on the same property. High transition counts between deed transfers and mortgages, for example, confirm the expected purchase-then-finance pattern. Unexpected pairings (forced sale signals following equity unlocks) may signal emerging distress.



{% horizontal_bar_chart
    data="transition_preview"
    y="source_display_name"
    x="transition_count"
    series="target_display_name"
    x_fmt="num0"
    title="Top Transitions from Transaction Propensity Instruments"
    y_sort="data"
/%}

{% table data="transition_preview" page_size=25 %}
    {% dimension value="source_display_name" title="From" /%}
    {% dimension value="target_display_name" title="To" /%}
    {% measure value="transition_count" title="Count" fmt="num0" /%}
    {% measure value="median_days_between" title="Median Days" fmt="num0" /%}
{% /table %}

## So What?

Is transaction activity accelerating or settling into a new baseline? The control charts above show whether the current month is statistically different from the recent pattern, or just noise.

If the Divergence Monitor shows equity unlocks and pressure signals converging, the Owner Pressure subcategory page has the severity breakdown and intervention queue. If a subcategory appears in Attention Now, Signal Detail provides instrument-level drill-through to identify what drove the spike.

## Signal Classification Reference

These tables show which instrument types are classified under each of the 7 Transaction Propensity subcategories, along with the three-axis signal weights (signal, distress, revenue) on a 1 to 10 scale. This is the logic that determines which instruments appear in this category.

```sql dim_transaction_propensity
-- Dimension reference: all instrument types classified as Transaction Propensity Intelligence
-- Source: dim_signal_category
-- Filter: category = 'Transaction Propensity Intelligence'
-- 93 rows across 7 subcategories
SELECT
    instrument_type
  , instrument_sub_type
  , subcategory
  , signal_weight
  , distress_weight
  , revenue_weight
FROM dim_signal_category
WHERE category = 'Transaction Propensity Intelligence'
ORDER BY subcategory, instrument_type, instrument_sub_type
```

### Transaction Timing Triggers

Deeds, assignments, and sales that mark ownership changes.

{% table data="dim_transaction_propensity" where="subcategory = 'Transaction Timing Triggers'" page_size=25 %}
    {% dimension value="instrument_type" title="Type" /%}
    {% dimension value="instrument_sub_type" title="Sub-Type" /%}
    {% measure value="signal_weight" title="Signal" fmt="num0" /%}
    {% measure value="distress_weight" title="Distress" fmt="num0" /%}
    {% measure value="revenue_weight" title="Revenue" fmt="num0" /%}
{% /table %}

### Equity Unlock Signals

Releases, satisfactions, and reconveyances that free collateral.

{% table data="dim_transaction_propensity" where="subcategory = 'Equity Unlock Signals'" page_size=25 %}
    {% dimension value="instrument_type" title="Type" /%}
    {% dimension value="instrument_sub_type" title="Sub-Type" /%}
    {% measure value="signal_weight" title="Signal" fmt="num0" /%}
    {% measure value="distress_weight" title="Distress" fmt="num0" /%}
    {% measure value="revenue_weight" title="Revenue" fmt="num0" /%}
{% /table %}

### Owner Pressure Signals

Tax liens, delinquencies, lis pendens, and foreclosure notices.

{% table data="dim_transaction_propensity" where="subcategory = 'Owner Pressure Signals'" page_size=25 %}
    {% dimension value="instrument_type" title="Type" /%}
    {% dimension value="instrument_sub_type" title="Sub-Type" /%}
    {% measure value="signal_weight" title="Signal" fmt="num0" /%}
    {% measure value="distress_weight" title="Distress" fmt="num0" /%}
    {% measure value="revenue_weight" title="Revenue" fmt="num0" /%}
{% /table %}

### Forced Sale Signals

Sheriff's deeds, tax sale certificates, and auction confirmations.

{% table data="dim_transaction_propensity" where="subcategory = 'Forced Sale Signals'" page_size=25 %}
    {% dimension value="instrument_type" title="Type" /%}
    {% dimension value="instrument_sub_type" title="Sub-Type" /%}
    {% measure value="signal_weight" title="Signal" fmt="num0" /%}
    {% measure value="distress_weight" title="Distress" fmt="num0" /%}
    {% measure value="revenue_weight" title="Revenue" fmt="num0" /%}
{% /table %}

### Life and Legal Triggers

Probate filings, death certificates, divorce decrees, and guardianships.

{% table data="dim_transaction_propensity" where="subcategory = 'Life and Legal Triggers'" page_size=25 %}
    {% dimension value="instrument_type" title="Type" /%}
    {% dimension value="instrument_sub_type" title="Sub-Type" /%}
    {% measure value="signal_weight" title="Signal" fmt="num0" /%}
    {% measure value="distress_weight" title="Distress" fmt="num0" /%}
    {% measure value="revenue_weight" title="Revenue" fmt="num0" /%}
{% /table %}

### Blocker Clearance Signals

Cancellations, terminations, and withdrawals that remove prior obstacles.

{% table data="dim_transaction_propensity" where="subcategory = 'Blocker Clearance Signals'" page_size=25 %}
    {% dimension value="instrument_type" title="Type" /%}
    {% dimension value="instrument_sub_type" title="Sub-Type" /%}
    {% measure value="signal_weight" title="Signal" fmt="num0" /%}
    {% measure value="distress_weight" title="Distress" fmt="num0" /%}
    {% measure value="revenue_weight" title="Revenue" fmt="num0" /%}
{% /table %}

### Deal Propensity Events

Option agreements and purchase contracts signaling active deals.

{% table data="dim_transaction_propensity" where="subcategory = 'Deal Propensity Events'" page_size=25 %}
    {% dimension value="instrument_type" title="Type" /%}
    {% dimension value="instrument_sub_type" title="Sub-Type" /%}
    {% measure value="signal_weight" title="Signal" fmt="num0" /%}
    {% measure value="distress_weight" title="Distress" fmt="num0" /%}
    {% measure value="revenue_weight" title="Revenue" fmt="num0" /%}
{% /table %}