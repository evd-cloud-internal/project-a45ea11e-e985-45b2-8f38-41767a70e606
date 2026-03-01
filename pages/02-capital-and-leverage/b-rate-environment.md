---
name: B Rate Environment
assetId: 8aeef822-b859-48f2-8208-da48efca9ee9
type: page
---

---
title: Rate Environment
---

# Where are rates headed?

Capital and Leverage Intelligence asks: "What is the capital position?" The rate environment determines who can refinance, who should hold, and how large the opportunity pipeline is at any given moment. This page tracks the Treasury-to-mortgage spread, historical rate regimes, and forward-looking scenarios that shape the size and urgency of the lending opportunity.

The spread between the 10-year Treasury yield and the 30-year fixed mortgage rate is the single best indicator of lending market conditions. When the spread compresses, lenders compete harder and borrowers get better terms. When it widens, credit tightens and refi pipelines shrink.

```sql current_snapshot
-- Most recent month with complete rate data from gold.rate_spread_analysis
SELECT
    rate_month
  , national_mortgage_30yr
  , treasury_10yr
  , spread
  , spread_12mo_avg
  , spread_percentile_rank
  , spread_status
FROM gold_rate_spread_analysis
ORDER BY rate_month DESC NULLS LAST
LIMIT 1
```

## Current Conditions

{% big_value
    data="current_snapshot"
    value="national_mortgage_30yr"
    title="30-Year Mortgage Rate"
    fmt="num2"
/%}

{% big_value
    data="current_snapshot"
    value="treasury_10yr"
    title="10-Year Treasury"
    fmt="num2"
/%}

{% big_value
    data="current_snapshot"
    value="spread"
    title="Current Spread"
    fmt="num2"
/%}

{% big_value
    data="current_snapshot"
    value="spread_status"
    title="Spread Regime"
/%}

> The spread is the gap between the 30-year mortgage rate and the 10-year Treasury yield. A spread below 1.50 is compressed (favorable for borrowers). Between 1.50 and 1.80 is the historical norm. Above 2.30 is stressed (lenders pricing in extra risk).

## Has the spread been here before?

```sql spread_history
-- Full spread history from gold.rate_spread_analysis
-- Used for line chart with regime reference areas
SELECT
    rate_month
  , spread
  , spread_12mo_avg
FROM gold_rate_spread_analysis
ORDER BY rate_month
```

{% line_chart
    data="spread_history"
    x="rate_month"
    y=["spread", "spread_12mo_avg"]
    y_fmt="num2"
    title="Treasury-to-Mortgage Spread (1971 to Present)"
    subtitle="Monthly spread with 12-month trailing average"
    chart_options={
        series_colors={
            "spread"="#006BA4"
            "spread_12mo_avg"="#898989"
        }
    }
    y_axis_options={
        fit_to_data=true
    }
%}
    {% reference_area
        y_min=0
        y_max=1.50
        label="Compressed"
        color="#3b82f6"
        area_options={ opacity=0.08 }
    /%}
    {% reference_area
        y_min=1.50
        y_max=1.80
        label="Normal"
        color="#22c55e"
        area_options={ opacity=0.08 }
    /%}
    {% reference_area
        y_min=1.80
        y_max=2.30
        label="Elevated"
        color="#f59e0b"
        area_options={ opacity=0.08 }
    /%}
    {% reference_area
        y_min=2.30
        y_max=5.00
        label="Stressed"
        color="#ef4444"
        area_options={ opacity=0.08 }
    /%}
{% /line_chart %}

The shaded bands show how much of the chart falls into each regime. Long stretches in the "Normal" band (1990s, mid-2010s) correspond to stable lending markets. The post-2022 spike into "Stressed" territory was the widest sustained spread since the early 1980s, and it took nearly three years to compress back toward historical norms.

```sql spread_regime_distribution
-- How many months in each spread regime across full history
-- pct_of_history is whole-number percentage (use num1, not pct1)
SELECT
    spread_status
  , COUNT(*) AS months_in_regime
  , ROUND(
        CAST(COUNT(*) AS Float64) * 100.0
        / (SELECT CAST(COUNT(*) AS Float64) FROM gold_rate_spread_analysis),
    1) AS pct_of_history
  , ROUND(MIN(spread), 2) AS min_spread
  , ROUND(MAX(spread), 2) AS max_spread
FROM gold_rate_spread_analysis
GROUP BY spread_status
ORDER BY
    CASE spread_status
        WHEN 'COMPRESSED' THEN 1
        WHEN 'NORMAL' THEN 2
        WHEN 'ELEVATED' THEN 3
        WHEN 'STRESSED' THEN 4
    END
```

{% table data="spread_regime_distribution" %}
    {% dimension value="spread_status" title="Regime" /%}
    {% measure value="months_in_regime" title="Months" fmt="num0" /%}
    {% measure value="pct_of_history" title="% of History" fmt="num1" /%}
    {% measure value="min_spread" title="Min Spread" fmt="num2" /%}
    {% measure value="max_spread" title="Max Spread" fmt="num2" /%}
{% /table %}

> The percentile rank of the current spread tells you where today sits relative to every month since 1971. A rank below 25 means the spread is tighter than three-quarters of all historical months. A rank above 75 means conditions are unusually wide.

## Treasury and mortgage rates since 1971

```sql rate_history
-- Dual-axis rate history from gold.rate_spread_analysis
SELECT
    rate_month
  , treasury_10yr
  , national_mortgage_30yr
FROM gold_rate_spread_analysis
ORDER BY rate_month
```

{% line_chart
    data="rate_history"
    x="rate_month"
    y=["national_mortgage_30yr", "treasury_10yr"]
    y_fmt="num1"
    title="30-Year Mortgage vs 10-Year Treasury"
    subtitle="Both rates move together; the gap between them is the spread"
    chart_options={
        series_colors={
            "national_mortgage_30yr"="#FF800E"
            "treasury_10yr"="#006BA4"
        }
    }
    y_axis_options={
        fit_to_data=true
    }
/%}

The two rates track each other closely because mortgage pricing is built on top of Treasury yields. The spread captures everything else: credit risk premiums, lender margins, MBS market conditions, and regulatory overhead. When the lines move apart, the lending market is under stress. When they converge, conditions are normalizing.

Guam's economy has historically been insulated from the mainland rate cycle. Military payroll and federal government spending provide a stable income floor, which means rate shocks that trigger distress waves on the mainland tend to shrink the refi pipeline here rather than produce defaults. The 2022 rate shock is the clearest example: the national 30-year rate nearly doubled, but Guam recorded no corresponding spike in foreclosure or distress filings.

## Where could rates go from here?

```sql scenario_data
-- 4 rate scenarios with trailing actuals for chart continuity
-- gold.rate_scenario_projections: trailing 24 months actual + 12 months projected
SELECT
    rate_month
  , scenario_label
  , projected_mortgage
  , is_actual
FROM gold_rate_scenario_projections
ORDER BY scenario_label, rate_month
```

```sql projection_boundary
-- Boundary between actual data and projections
-- Used for vertical reference line on scenario chart
-- Label column required: data-driven reference_line interprets label= as column name
SELECT
    MAX(rate_month) AS boundary_date
  , 'Projections start' AS boundary_label
FROM gold_rate_scenario_projections
WHERE is_actual = 1
```

{% line_chart
    data="scenario_data"
    x="rate_month"
    y="projected_mortgage"
    series="scenario_label"
    y_fmt="num2"
    title="Projected 30-Year Mortgage Rate by Scenario"
    subtitle="24 months actual history flowing into 12 months projected"
    chart_options={
        series_colors={
            "A: Rates Fall"="#22c55e"
            "B: Soft Landing"="#3b82f6"
            "C: Status Quo"="#6b7280"
            "D: Stress"="#ef4444"
        }
    }
    y_axis_options={
        fit_to_data=true
    }
%}
    {% reference_line
        data="projection_boundary"
        x="boundary_date"
        label="boundary_label"
        color="#6b7280"
        line_options={ type="dashed" width=1 }
        label_options={ position="above_center" }
    /%}
{% /line_chart %}

The four scenarios bracket the plausible range. All four share the same 24 months of actual data on the left side of the chart; they fan out at the projection boundary.

```sql scenario_endpoints
-- 12-month-out endpoint for each scenario
-- Shows where each path leads by the end of the projection window
SELECT
    scenario_label AS scenario
  , projected_treasury AS treasury_rate
  , projected_spread AS spread
  , projected_mortgage AS mortgage_rate
FROM gold_rate_scenario_projections
WHERE is_actual = 0
  AND rate_month = (
      SELECT MAX(rate_month)
      FROM gold_rate_scenario_projections
      WHERE is_actual = 0
  )
ORDER BY scenario_label
```

{% table data="scenario_endpoints" %}
    {% dimension value="scenario" title="Scenario" /%}
    {% measure value="treasury_rate" title="Treasury (12mo)" fmt="num2" /%}
    {% measure value="spread" title="Spread" fmt="num2" /%}
    {% measure value="mortgage_rate" title="Mortgage Rate (12mo)" fmt="num2" /%}
{% /table %}

> **A: Rates Fall** assumes the Treasury drops 25 basis points per quarter and the spread normalizes to 1.70. **B: Soft Landing** holds the Treasury flat with spread normalization. **C: Status Quo** holds everything at current levels. **D: Stress** adds 15 basis points per quarter to the Treasury with the spread widening to 2.50. Each scenario produces a different refi-eligible population on the Rate Gap and Refi page.

## So what?

Is the spread compressing or widening? The answer determines how many active mortgages cross the refinance threshold. The Rate Gap and Refi page shows which vintage mortgages qualify under current conditions and how that population changes under each scenario.