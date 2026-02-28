---
name: A Mortgage Book
assetId: 8ce059aa-b27a-402a-9e43-7af99446aea6
type: page
---

---
title: What is the capital position?
---

Mortgages, modifications, subordinations, bonds, and promissory notes: 17 instrument types that reveal how debt is structured, restructured, and traded across Guam's real estate market. This is the largest revenue-weighted signal family in the platform (signal=10, revenue=10), and the one most directly tied to lending product opportunities.

_Capital & Leverage Intelligence: mortgages, modifications, subordinations, bonds, and promissory notes._

## Data Health

```sql health_amounts
-- Capital & Leverage: dollar amount coverage
-- Source: audit_amount_data_quality (category column verified at 01_dimensions line 174)
SELECT
    SUM(total_count) AS total_instruments
  , ROUND(SUM(has_any_amount_count) * 100.0 / SUM(total_count), 1) AS pct_with_amount
FROM audit_amount_data_quality
WHERE category = 'Capital & Leverage Intelligence'
```

```sql health_lag
-- Capital & Leverage: median recording lag (weighted by instrument count)
-- Source: audit_recording_lag_outliers (category column verified at 01_dimensions line 174)
SELECT
    ROUND(
      SUM(total_count * median_days_to_record) * 1.0
      / SUM(total_count), 0
    ) AS median_lag_days
FROM audit_recording_lag_outliers
WHERE category = 'Capital & Leverage Intelligence'
```

> **{% value data="health_amounts" value="total_instruments" fmt="num0k" /%}** instruments in this category. **{% value data="health_amounts" value="pct_with_amount / 100" fmt="pct0" /%}** carry a dollar amount. Median recording lag: **{% value data="health_lag" value="median_lag_days" fmt="num0" /%} days**.

## Is capital activity stable or shifting?

```sql volume_control
-- Capital & Leverage: monthly volume with XmR control chart limits
-- Source: gold_signal_velocity_trends (category = 'Capital & Leverage Intelligence', verified 01_dimensions line 174)
-- XmR: mean + 3 * (avg_moving_range / 1.128). Statistical constant 1.128 for consecutive pairs.
WITH monthly AS (
    SELECT
        record_month
      , SUM(current_month_count) AS volume
    FROM gold_signal_velocity_trends
    WHERE category = 'Capital & Leverage Intelligence'
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
SELECT
    with_mr.record_month
  , with_mr.volume
  , ROUND(AVG(with_mr.volume) OVER (
      ORDER BY with_mr.record_month
      ROWS BETWEEN 5 PRECEDING AND CURRENT ROW), 0) AS volume_6mo_avg
  , ROUND(stats.mean_vol, 0) AS mean_line
  , ROUND(stats.mean_vol + 3 * (stats.avg_mr / 1.128), 0) AS ucl
  , GREATEST(0, ROUND(stats.mean_vol - 3 * (stats.avg_mr / 1.128), 0)) AS lcl
FROM with_mr
CROSS JOIN stats
-- Trim leading partial month (silver.monthly_signal_velocity 3-year window edge)
WHERE with_mr.record_month > (SELECT MIN(record_month) FROM monthly)
ORDER BY with_mr.record_month
```

```sql volume_series
-- Unpivot volume data for two-series line chart
-- Monthly (navy) and 6-Mo Average (gray) per colorblind-safe palette
SELECT record_month, 'Monthly' AS series, volume AS value
FROM {{volume_control}}

UNION ALL

SELECT record_month, '6-Mo Average' AS series, volume_6mo_avg AS value
FROM {{volume_control}}
```

```sql volume_limits
-- XmR reference lines: Mean, UCL, LCL (constant across time)
SELECT mean_line AS value, 'Mean' AS label FROM {{volume_control}} LIMIT 1
UNION ALL
SELECT ucl AS value, 'UCL' AS label FROM {{volume_control}} LIMIT 1
UNION ALL
SELECT lcl AS value, 'LCL' AS label FROM {{volume_control}} LIMIT 1
```

{% line_chart
    data="volume_series"
    x="record_month"
    y="value"
    series="series"
    y_fmt="num0"
    title="Monthly Capital & Leverage Instruments"
    chart_options={
        series_colors={
            "Monthly"="#006BA4"
            "6-Mo Average"="#898989"
        }
    }
%}
    {% reference_line
        data="volume_limits"
        y="value"
        label="label"
        color="#cccccc"
        line_options={type="dashed"}
    /%}
{% /line_chart %}

```sql dollar_control
-- Capital & Leverage: monthly dollar volume with XmR control chart limits
-- Source: gold_signal_velocity_trends (category = 'Capital & Leverage Intelligence', verified 01_dimensions line 174)
WITH monthly AS (
    SELECT
        record_month
      , SUM(current_month_amount) AS dollars
    FROM gold_signal_velocity_trends
    WHERE category = 'Capital & Leverage Intelligence'
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
SELECT
    with_mr.record_month
  , with_mr.dollars
  , ROUND(AVG(with_mr.dollars) OVER (
      ORDER BY with_mr.record_month
      ROWS BETWEEN 5 PRECEDING AND CURRENT ROW), 0) AS dollars_6mo_avg
  , ROUND(stats.mean_dol, 0) AS mean_line
  , ROUND(stats.mean_dol + 3 * (stats.avg_mr / 1.128), 0) AS ucl
  , GREATEST(0, ROUND(stats.mean_dol - 3 * (stats.avg_mr / 1.128), 0)) AS lcl
FROM with_mr
CROSS JOIN stats
WHERE with_mr.record_month > (SELECT MIN(record_month) FROM monthly)
ORDER BY with_mr.record_month
```

```sql dollar_series
-- Unpivot dollar data for two-series line chart
SELECT record_month, 'Monthly' AS series, dollars AS value
FROM {{dollar_control}}

UNION ALL

SELECT record_month, '6-Mo Average' AS series, dollars_6mo_avg AS value
FROM {{dollar_control}}
```

```sql dollar_limits
-- XmR reference lines for dollar chart
SELECT mean_line AS value, 'Mean' AS label FROM {{dollar_control}} LIMIT 1
UNION ALL
SELECT ucl AS value, 'UCL' AS label FROM {{dollar_control}} LIMIT 1
UNION ALL
SELECT lcl AS value, 'LCL' AS label FROM {{dollar_control}} LIMIT 1
```

{% line_chart
    data="dollar_series"
    x="record_month"
    y="value"
    series="series"
    y_fmt="usd0m"
    title="Monthly Capital & Leverage Dollar Volume"
    chart_options={
        series_colors={
            "Monthly"="#FF800E"
            "6-Mo Average"="#898989"
        }
    }
%}
    {% reference_line
        data="dollar_limits"
        y="value"
        label="label"
        color="#cccccc"
        line_options={type="dashed"}
    /%}
{% /line_chart %}

Control limits calculated using XmR method (see Control Chart Method in appendix).

## Anything out of the ordinary?

```sql attention_now
-- XmR out-of-control detection for Capital & Leverage subcategories
-- Source: gold_signal_velocity_trends (category = 'Capital & Leverage Intelligence', verified 01_dimensions line 174)
-- Note: this category has only 1 subcategory. Table will have 0 or 1 rows.
WITH monthly AS (
    SELECT
        record_month
      , subcategory
      , SUM(current_month_count) AS volume
    FROM gold_signal_velocity_trends
    WHERE category = 'Capital & Leverage Intelligence'
    GROUP BY record_month, subcategory
)
, with_mr AS (
    SELECT
        record_month
      , subcategory
      , volume
      , ABS(volume - LAG(volume) OVER (PARTITION BY subcategory ORDER BY record_month)) AS moving_range
    FROM monthly
)
, limits AS (
    SELECT
        subcategory
      , AVG(volume) AS mean_vol
      , AVG(volume) + 3 * (AVG(moving_range) / 1.128) AS ucl
      , GREATEST(0, AVG(volume) - 3 * (AVG(moving_range) / 1.128)) AS lcl
    FROM with_mr
    GROUP BY subcategory
)
, eval AS (
    -- Last complete month (skip current partial)
    SELECT DISTINCT record_month
    FROM gold_signal_velocity_trends
    ORDER BY record_month DESC
    LIMIT 1 OFFSET 1
)
SELECT
    with_mr.subcategory
  , with_mr.volume AS current_volume
  , ROUND(limits.mean_vol, 0) AS mean
  , ROUND(limits.ucl, 0) AS ucl
  , ROUND(limits.lcl, 0) AS lcl
  , CASE
      WHEN with_mr.volume > limits.ucl THEN 'ABOVE UCL'
      WHEN with_mr.volume < limits.lcl THEN 'BELOW LCL'
    END AS status
FROM with_mr
JOIN limits ON with_mr.subcategory = limits.subcategory
JOIN eval ON with_mr.record_month = eval.record_month
WHERE with_mr.volume > limits.ucl
   OR with_mr.volume < limits.lcl
```

{% if data="attention_now" %}

{% table data="attention_now" page_size=5 %}
    {% measure value="subcategory" title="Subcategory" /%}
    {% measure value="current_volume" title="Current" fmt="num0" /%}
    {% measure value="mean" title="Mean" fmt="num0" /%}
    {% measure value="ucl" title="UCL" fmt="num0" /%}
    {% measure value="status" title="Status" /%}
{% /table %}

A subcategory outside its control limits means the last complete month was statistically unusual. The Signal Detail page in Signal Patterns shows which instruments drove the spike or dip.

{% /if %}
{% else %}

All subcategories in this category are within normal variation for the last complete month.

{% /else %}

## This month vs last month

{% big_value
    data="gold_signal_velocity_trends"
    value="sum(current_month_count)"
    title="Instruments"
    fmt="num0"
    where="category = 'Capital & Leverage Intelligence'"
    date_range={
        date="record_month"
        range="previous month"
    }
    comparison={
        compare_vs="prior period"
        pct_fmt="pct0"
    }
/%}

{% big_value
    data="gold_signal_velocity_trends"
    value="sum(current_month_amount)"
    title="Dollar Volume"
    fmt="usd0m"
    where="category = 'Capital & Leverage Intelligence'"
    date_range={
        date="record_month"
        range="previous month"
    }
    comparison={
        compare_vs="prior period"
        pct_fmt="pct0"
    }
/%}

## The mortgage book

```sql refi_dashboard
-- Mortgage product breakdown with refi window and eligibility counts
-- Source: gold_refi_opportunity_dashboard (columns verified from 07_gold lines 224-280)
SELECT
    mortgage_type_group AS product_group
  , SUM(total_mortgages) AS total_mortgages
  , SUM(total_principal) AS total_principal
  , SUM(refi_window_count) AS refi_window
  , SUM(refi_window_volume) AS refi_window_volume
  , SUM(va_irrrl_ready) AS va_irrrl
  , SUM(fha_streamline_ready) AS fha_streamline
  , SUM(heloc_reset_window) AS heloc_reset
  , SUM(private_conversion_ready) AS private_conversion
  , SUM(const_perm_ready) AS construction_perm
  , SUM(equity_rich_count) AS equity_rich
FROM gold_refi_opportunity_dashboard
GROUP BY mortgage_type_group
ORDER BY SUM(total_principal) DESC NULLS LAST
```

{% table data="refi_dashboard" page_size=200 order="total_principal desc" %}
    {% measure value="product_group" title="Product Group" /%}
    {% measure value="total_mortgages" title="Total" fmt="num0" /%}
    {% measure value="total_principal" title="Principal" fmt="usd0m" /%}
    {% measure value="refi_window" title="Refi Window" fmt="num0" /%}
    {% measure value="va_irrrl" title="VA IRRRL" fmt="num0" /%}
    {% measure value="fha_streamline" title="FHA Stream." fmt="num0" /%}
    {% measure value="heloc_reset" title="HELOC Reset" fmt="num0" /%}
    {% measure value="equity_rich" title="Equity Rich" fmt="num0" /%}
{% /table %}

The refi window column counts active mortgages originated 24 to 60 months ago. VA IRRRL, FHA Streamline, and HELOC Reset are government program eligibility flags. Equity Rich means the property has an estimated LTV under 60%. The Rate Gap & Refi page in this folder breaks these numbers down by rate opportunity tier and provides contact-ready work queues.

## Where does the rate gap stand?

```sql rate_gap_summary
-- Rate gap portfolio by opportunity tier (rollup level, all product types combined)
-- Source: gold_rate_gap_portfolio_summary (columns verified from 07_gold lines 725-769)
-- 'ALL_TYPES' literal from line 731: GROUPING SETS rollup produces this value
SELECT
    rate_gap_opportunity
  , active_count
  , ROUND(active_principal, 0) AS active_principal
  , wtd_avg_origination_rate
  , wtd_avg_rate_gap
  , refi_sweet_spot_count
  , govt_streamline_count
  , ROUND(pct_of_active_principal, 0) AS pct_of_portfolio
FROM gold_rate_gap_portfolio_summary
WHERE detail_level = 'ALL_TYPES'
ORDER BY
    CASE rate_gap_opportunity
        WHEN 'STRONG_REFI' THEN 1
        WHEN 'MILD_REFI' THEN 2
        WHEN 'NEUTRAL' THEN 3
        WHEN 'RETENTION' THEN 4
        WHEN 'STRONG_RETENTION' THEN 5
        ELSE 6
    END
```

{% table data="rate_gap_summary" page_size=200 %}
    {% measure value="rate_gap_opportunity" title="Tier" /%}
    {% measure value="active_count" title="Active Mortgages" fmt="num0" /%}
    {% measure value="active_principal" title="Active Principal" fmt="usd0m" /%}
    {% measure value="wtd_avg_origination_rate" title="Avg Orig. Rate" fmt="pct0" /%}
    {% measure value="wtd_avg_rate_gap" title="Avg Rate Gap" fmt="num0" /%}
    {% measure value="pct_of_portfolio" title="% of Portfolio" fmt="pct0" /%}
{% /table %}

STRONG_REFI borrowers originated at rates well above the current Guam effective rate. STRONG_RETENTION borrowers are at rates below current market, meaning a competitor refinance is unlikely. The Rate Gap & Refi page has the full breakdown by product type and the contact-ready queues.

## How has the product mix evolved?

```sql market_share
-- Mortgage product mix by vintage year (2018+)
-- Source: gold_mortgage_market_share_trends (columns verified from 07_gold lines 289-313)
SELECT
    vintage_year
  , mortgage_type_group
  , mortgage_count
  , total_principal
  , pct_of_count
  , pct_of_volume
FROM gold_mortgage_market_share_trends
ORDER BY vintage_year, mortgage_type_group
```

{% table data="market_share" page_size=200 order="vintage_year desc" %}
    {% measure value="vintage_year" title="Vintage" /%}
    {% measure value="mortgage_type_group" title="Product" /%}
    {% measure value="mortgage_count" title="Count" fmt="num0" /%}
    {% measure value="total_principal" title="Principal" fmt="usd0m" /%}
    {% measure value="pct_of_count" title="% Count" fmt="pct0" /%}
    {% measure value="pct_of_volume" title="% Volume" fmt="pct0" /%}
{% /table %}

Which product types are growing share? Is government-backed lending expanding or contracting? The Rate Environment page shows the macro rate context that drives these shifts.

## Who has equity, and how much?

```sql ltv_dashboard
-- LTV distribution across active mortgage portfolio
-- Source: gold_ltv_equity_dashboard (columns verified from 07_gold lines 678-706)
-- LTV tier values verified from 07_gold lines 699-703
SELECT
    ltv_tier
  , SUM(mortgage_count) AS mortgage_count
  , SUM(total_principal) AS total_principal
  , ROUND(SUM(mortgage_count * avg_ltv) / NULLIF(SUM(mortgage_count), 0), 0) AS avg_ltv
  , ROUND(SUM(mortgage_count * avg_estimated_equity) / NULLIF(SUM(mortgage_count), 0), 0) AS avg_equity
  , SUM(equity_rich_count) AS equity_rich_count
  , SUM(underwater_count) AS underwater_count
  , ROUND(SUM(pct_of_portfolio * mortgage_count) / NULLIF(SUM(mortgage_count), 0), 0) AS pct_of_portfolio
FROM gold_ltv_equity_dashboard
GROUP BY ltv_tier
ORDER BY
    CASE ltv_tier
        WHEN 'UNDER_60' THEN 1
        WHEN '60_80' THEN 2
        WHEN '80_100' THEN 3
        WHEN 'OVER_100' THEN 4
        ELSE 5
    END
```

{% table data="ltv_dashboard" page_size=200 %}
    {% measure value="ltv_tier" title="LTV Tier" /%}
    {% measure value="mortgage_count" title="Mortgages" fmt="num0" /%}
    {% measure value="total_principal" title="Principal" fmt="usd0m" /%}
    {% measure value="avg_ltv" title="Avg LTV" fmt="num0" /%}
    {% measure value="avg_equity" title="Avg Equity" fmt="usd0" /%}
    {% measure value="equity_rich_count" title="Equity Rich" fmt="num0" /%}
    {% measure value="underwater_count" title="Underwater" fmt="num0" /%}
{% /table %}

UNDER_60 borrowers are prime refinance and HELOC targets: significant equity, strong collateral position. OVER_100 are counseling candidates. The Equity Unlocks page in Market Activity tracks the instruments (satisfactions, reconveyances) that signal equity becoming available.

## Do capital events and forced sales move together?

```sql divergence
-- Divergence monitor: Capital & Leverage volume vs Forced Sale volume
-- Source: gold_signal_velocity_trends
-- 'Capital & Leverage Intelligence' verified 01_dimensions line 174
-- 'Forced Sale Signals' verified 01_dimensions line 264
SELECT
    record_month
  , subcategory AS series
  , SUM(current_month_count) AS volume
FROM gold_signal_velocity_trends
WHERE subcategory IN ('Capital & Leverage Intelligence', 'Forced Sale Signals')
  -- Trim leading partial month
  AND record_month > (
    SELECT MIN(record_month) FROM gold_signal_velocity_trends
  )
GROUP BY record_month, subcategory
ORDER BY record_month
```

{% line_chart
    data="divergence"
    x="record_month"
    y="volume"
    series="series"
    y_fmt="num0"
    title="Capital Activity vs Forced Sales"
    chart_options={
        series_colors={
            "Capital & Leverage Intelligence"="#1F77B4"
            "Forced Sale Signals"="#E86B00"
        }
    }
/%}

When capital activity rises but forced sales stay flat, borrowers are restructuring voluntarily. When forced sales spike while capital activity is steady, the market is producing involuntary exits. Divergence between the two lines is the signal; convergence means the forces are in balance.

## What follows a capital event?

```sql transitions
-- Top 15 transitions originating from Capital & Leverage instruments
-- Source: silver_instrument_transitions_combined (columns verified from 06_silver lines 1441-1477)
-- Filtered to property-linked transitions for strongest signal
SELECT
    source_subcategory
  , target_subcategory
  , transition_count
  , median_days_between
FROM silver_instrument_transitions_combined
WHERE source_subcategory = 'Capital & Leverage Intelligence'
  AND linkage_type = 'property'
ORDER BY transition_count DESC
LIMIT 15
```

{% horizontal_bar_chart
    data="transitions"
    y="target_subcategory"
    x="transition_count"
    x_fmt="num0"
    y_sort="data"
    title="What happens next on the same property?"
/%}

{% table data="transitions" page_size=15 order="transition_count desc" %}
    {% measure value="target_subcategory" title="Next Signal" /%}
    {% measure value="transition_count" title="Count" fmt="num0" /%}
    {% measure value="median_days_between" title="Median Days" fmt="num0" /%}
{% /table %}

Properties that record a capital event (mortgage, modification, subordination) often produce follow-on signals. The median days column shows how quickly those follow-on events typically appear. The Instrument Flows page in Signal Patterns has the full transition matrix across all categories.

## So what?

Is the monthly volume trend stable, or has something shifted? The control charts separate real change from noise. If capital activity is climbing while the rate environment compresses (check the Rate Environment page in this folder), borrowers are responding to improved conditions. If it is falling while rates are still elevated, the market may be waiting.

## Reference: Mortgage Product Types

```sql dim_mortgage_type
-- Mortgage product grouping reference table
-- Source: dim_mortgage_type (14 rows, verified from 01_dimensions lines 399-419)
SELECT
    mortgage_type
  , mortgage_type_group
FROM dim_mortgage_type
ORDER BY mortgage_type_group, mortgage_type
```

{% table data="dim_mortgage_type" page_size=200 %}
    {% measure value="mortgage_type_group" title="Product Group" /%}
    {% measure value="mortgage_type" title="Mortgage Type Code" /%}
{% /table %}

## Reference: Capital Instrument Types

```sql dim_capital
-- Capital instrument type reference table
-- Source: dim_capital_instrument_type (13 rows, verified from 01_dimensions lines 1679-1699)
SELECT
    instrument_type
  , capital_instrument_type
FROM dim_capital_instrument_type
ORDER BY capital_instrument_type
```

{% table data="dim_capital" page_size=200 %}
    {% measure value="capital_instrument_type" title="Capital Type" /%}
    {% measure value="instrument_type" title="Instrument Type Code" /%}
{% /table %}

## Reference: Signal Classification

```sql dim_signal_cat
-- Signal category classification for Capital & Leverage instruments
-- Source: dim_signal_category (17 rows where category = 'Capital & Leverage Intelligence')
-- Category literal verified at 01_dimensions line 174
SELECT
    instrument_type
  , instrument_sub_type
  , signal_weight
  , distress_weight
  , revenue_weight
FROM dim_signal_category
WHERE category = 'Capital & Leverage Intelligence'
ORDER BY instrument_type
```

{% table data="dim_signal_cat" page_size=200 %}
    {% measure value="instrument_type" title="Instrument Type" /%}
    {% measure value="instrument_sub_type" title="Sub-Type" /%}
    {% measure value="signal_weight" title="Signal Wt" fmt="num0" /%}
    {% measure value="distress_weight" title="Distress Wt" fmt="num0" /%}
    {% measure value="revenue_weight" title="Revenue Wt" fmt="num0" /%}
{% /table %}