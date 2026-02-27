---
name: Don't Delete This Page
assetId: d428acbd-cb2e-4231-b1f8-75f650a43ad2
type: page
---

---
title: Beacons
---

<!-- ================================================================
     Beacons: Executive Summary (Page 1)
     Source views: gold_signal_velocity_trends, gold_yearly_instrument_summary,
                   audit_signal_category_coverage, audit_bridge_relationship_coverage,
                   audit_mortgage_eligibility_flags, audit_data_freshness,
                   dim_signal_category
     Page type: Executive (whole-number formatting: pct0, num0, usd0m)
     Architecture plan: lines 176-191
     ================================================================ -->

# Beacons

Beacons transforms Guam Department of Land Management recorded instruments into predictive lending intelligence.

**How it works:**
- REALFacts captures every recorded instrument from DLM (the raw data)
- Beacons classifies those instruments into signal families
- Each instrument is scored on three axes: signal strength, revenue potential, and distress severity
- Scored signals feed prioritized work queues for lending teams

**Five questions organize the platform:**
- Who will transact next?
- Will this deal close?
- What is the capital position?
- Where is the market moving?
- Where are the compliance risks?

Each question maps to a category of signal. Each category has its own detail page accessible from the sidebar.

---

## Platform Snapshot

```sql total_instruments
-- Total instruments processed by the platform
-- Source: audit.signal_category_coverage (all rows, all statuses)
SELECT SUM(instrument_count) AS value
FROM audit_signal_category_coverage
```

```sql classification_pct
-- Percentage of instruments mapped to a signal family
-- Source: audit.signal_category_coverage, filtered to MAPPED status
-- Divided by 100 so Evidence pct0 format renders "97%"
SELECT
    ROUND(
        SUM(CASE WHEN mapping_status = 'MAPPED' THEN instrument_count ELSE 0 END) * 1.0
        / SUM(instrument_count),
        3
    ) AS value
FROM audit_signal_category_coverage
```

```sql linked_properties
-- Instruments linked to a legal unit (property) through bridge resolution
-- Source: audit.bridge_relationship_coverage (single-row view)
SELECT instruments_with_legal_unit AS value
FROM audit_bridge_relationship_coverage
```

```sql active_mortgages
-- Mortgages estimated as currently active based on term assumptions
-- Source: audit.mortgage_eligibility_flags (single-row view)
SELECT active_count AS value
FROM audit_mortgage_eligibility_flags
```

```sql data_freshness
-- Most recent recording date and staleness indicator
-- Source: audit.data_freshness (single-row view)
-- days_since_last_record surfaced for pipeline health monitoring (7-day threshold)
SELECT
    CAST(most_recent_record AS DATE) AS value,
    days_since_last_record
FROM audit_data_freshness
```

{% row %}
    {% big_value data="total_instruments" value="value" title="Total Instruments" fmt="num0k" /%}
    {% big_value data="classification_pct" value="value" title="Classification Coverage" fmt="pct0" /%}
    {% big_value data="linked_properties" value="value" title="Property-Linked Instruments" fmt="num0k" /%}
    {% big_value data="active_mortgages" value="value" title="Active Mortgages" fmt="num0k" /%}
    {% big_value data="data_freshness" value="value" title="Most Recent Recording" /%}
    {% big_value data="data_freshness" value="days_since_last_record" title="Days Since Last Record" fmt="num0" /%}
{% /row %}

> **Why this matters:** These numbers define the platform's analytical reach.  
- Classification coverage determines what percentage of the recorded universe produces signals.  
- Property linkage determines how many instruments can be traced to a physical asset.  
- Active mortgages define the addressable refinance market.  
- The most recent recording date confirms data freshness. If days since last record exceeds 7, investigate the pipeline before trusting the analytical pages.  
- If any of these numbers changes dramatically, the data pipeline needs investigation.

---

## Signal Families

Beacons organizes recorded instruments into 13 signal subcategories across 5 analytical categories. Every subcategory name used on this page and throughout the platform maps to the instrument types described below.

```sql signal_families
-- Reference table: all subcategories with plain English translations
-- Source: dim.signal_category (165 rows), distinct subcategory/category pairs
-- Subcategory literals: 01_dimensions INSERTs (lines 162-392)
-- Plain English translations: dashboard architecture plan v2 (lines 59-71)
-- Excludes Metadata (administrative filings with no signal value)
SELECT
    category,
    subcategory,
    CASE subcategory
        WHEN 'Equity Unlock Signals' THEN 'Releases, satisfactions, and reconveyances that free collateral'
        WHEN 'Owner Pressure Signals' THEN 'Tax liens, delinquencies, lis pendens, and foreclosure notices'
        WHEN 'Forced Sale Signals' THEN 'Sheriff''s deeds, tax sale certificates, and auction confirmations'
        WHEN 'Blocker Clearance Signals' THEN 'Cancellations, terminations, and withdrawals that remove prior obstacles'
        WHEN 'Transaction Timing Triggers' THEN 'Deeds, assignments, and sales that mark ownership changes'
        WHEN 'Life and Legal Triggers' THEN 'Probate filings, death certificates, divorce decrees, and guardianships'
        WHEN 'Deal Propensity Events' THEN 'Option agreements and purchase contracts signaling active deals'
        WHEN 'Capital & Leverage Intelligence' THEN 'Mortgages, modifications, subordinations, bonds, and promissory notes'
        WHEN 'Title and Ownership Complexity' THEN 'Quitclaim deeds, corrections, affidavits, and name changes'
        WHEN 'Deal Complexity Events' THEN 'Amendments, addenda, extensions, and escrow complications'
        WHEN 'Supply Signals' THEN 'New construction permits, subdivision plats, and condo declarations'
        WHEN 'Regulatory, Risk & Claims Intelligence' THEN 'Government liens, environmental filings, and regulatory actions'
        WHEN 'Liens and Encumbrances' THEN 'Mechanic''s liens, judgment liens, and federal tax liens'
        ELSE subcategory
    END AS what_it_tracks,
    COUNT(*) AS instrument_types_classified
FROM dim_signal_category
WHERE category != 'Metadata'
GROUP BY category, subcategory
ORDER BY
    CASE category
        WHEN 'Transaction Propensity Intelligence' THEN 1
        WHEN 'Transaction Feasibility & Close Intelligence' THEN 2
        WHEN 'Capital & Leverage Intelligence' THEN 3
        WHEN 'Market Dynamics Intelligence' THEN 4
        WHEN 'Regulatory, Risk & Claims Intelligence' THEN 5
    END,
    subcategory
```

{% table data="signal_families" page_size=200 %}
    {% dimension value="category" /%}
    {% dimension value="subcategory" /%}
    {% dimension value="what_it_tracks" /%}
    {% measure value="instrument_types_classified" fmt="num0" /%}
{% /table %}

> **How to read this table:** The "what it tracks" column describes the real-world documents behind each signal name. When a subcategory appears in a chart or alert elsewhere on this page, refer back here. The "instrument types classified" column shows how many distinct document types feed each subcategory.

---

## Attention Now

Subcategories where the most recent complete month fell outside statistical control limits. If this table is empty, all subcategories are within normal variation. Control limits use the XmR method (see appendix).

```sql attention_now
-- XmR out-of-control detection: subcategories outside 3-sigma limits
-- Source: gold.signal_velocity_trends (full 3-year window)
-- Subcategory values are dim-driven (source: dim.signal_category INSERTs in 01_dimensions)
-- Gold may show fewer subcategories if some have aged out of the 3-year silver window
-- Evaluates: second-most-recent month (offset 1, most recent complete month)
-- Statistical constant: 1.128 = d2 for n=2, XmR method
WITH monthly_counts AS (
    SELECT
        record_month,
        category,
        subcategory,
        current_month_count
    FROM gold_signal_velocity_trends
    WHERE category != 'Metadata'
),

with_moving_range AS (
    SELECT
        record_month,
        category,
        subcategory,
        current_month_count,
        ABS(current_month_count - LAG(current_month_count) OVER (
            PARTITION BY subcategory ORDER BY record_month
        )) AS moving_range
    FROM monthly_counts
),

control_limits AS (
    SELECT
        subcategory,
        category,
        AVG(current_month_count) AS mean_count,
        AVG(moving_range) AS avg_moving_range,
        AVG(current_month_count) + 3 * (AVG(moving_range) / 1.128) AS ucl,
        GREATEST(0, AVG(current_month_count) - 3 * (AVG(moving_range) / 1.128)) AS lcl
    FROM with_moving_range
    GROUP BY subcategory, category
),

evaluation_month AS (
    SELECT DISTINCT record_month
    FROM monthly_counts
    ORDER BY record_month DESC
    LIMIT 1 OFFSET 1
),

evaluation_values AS (
    SELECT
        subcategory,
        category,
        current_month_count AS eval_count,
        record_month AS eval_month
    FROM monthly_counts
    WHERE record_month = (SELECT record_month FROM evaluation_month)
)

SELECT
    evaluation_values.category,
    evaluation_values.subcategory,
    CASE evaluation_values.subcategory
        WHEN 'Equity Unlock Signals' THEN 'Releases, satisfactions, reconveyances'
        WHEN 'Owner Pressure Signals' THEN 'Tax liens, delinquencies, lis pendens'
        WHEN 'Forced Sale Signals' THEN 'Foreclosure deeds, tax deeds, auctions'
        WHEN 'Blocker Clearance Signals' THEN 'Cancellations, terminations, withdrawals'
        WHEN 'Transaction Timing Triggers' THEN 'Deeds, assignments, sales'
        WHEN 'Life and Legal Triggers' THEN 'Probate, death certificates, divorce'
        WHEN 'Deal Propensity Events' THEN 'Options, purchase contracts'
        WHEN 'Capital & Leverage Intelligence' THEN 'Mortgages, modifications, subordinations'
        WHEN 'Title and Ownership Complexity' THEN 'Quitclaims, corrections, affidavits'
        WHEN 'Deal Complexity Events' THEN 'Amendments, addenda, extensions'
        WHEN 'Supply Signals' THEN 'Construction permits, plats, condo declarations'
        WHEN 'Regulatory, Risk & Claims Intelligence' THEN 'Government liens, regulatory actions'
        WHEN 'Liens and Encumbrances' THEN 'Mechanic''s liens, judgment liens, tax liens'
        ELSE evaluation_values.subcategory
    END AS what_it_tracks,
    CAST(evaluation_values.eval_month AS VARCHAR) AS eval_month,
    evaluation_values.eval_count,
    ROUND(control_limits.mean_count, 0) AS mean_count,
    ROUND(control_limits.ucl, 0) AS upper_limit,
    ROUND(control_limits.lcl, 0) AS lower_limit,
    CASE
        WHEN evaluation_values.eval_count > control_limits.ucl THEN 'ABOVE UCL'
        WHEN evaluation_values.eval_count < control_limits.lcl THEN 'BELOW LCL'
    END AS signal_status
FROM evaluation_values
JOIN control_limits ON evaluation_values.subcategory = control_limits.subcategory
WHERE evaluation_values.eval_count > control_limits.ucl
   OR evaluation_values.eval_count < control_limits.lcl
ORDER BY ABS(evaluation_values.eval_count - control_limits.mean_count) DESC
```

{% table data="attention_now" page_size=200 %}
    {% dimension value="category" /%}
    {% dimension value="subcategory" /%}
    {% dimension value="what_it_tracks" title="What It Tracks" /%}
    {% dimension value="eval_month" title="Month" /%}
    {% measure value="eval_count" title="Eval Count" fmt="num0" /%}
    {% measure value="mean_count" title="Mean" fmt="num0" /%}
    {% measure value="upper_limit" title="UCL" fmt="num0" /%}
    {% measure value="lower_limit" title="LCL" fmt="num0" /%}
    {% dimension value="signal_status" title="Status" /%}
{% /table %}

> **So what?**  
- Empty table = every subcategory is within its historical range.  
- A row appearing here means that subcategory's most recent complete month fell outside three standard deviations from its mean.  
- That does not automatically mean something is wrong. Is the shift seasonal? Did a batch of recordings land late? Or is there a genuine change in market behavior?  
- The "what it tracks" column tells you what kinds of documents drove the anomaly.  
- Check the relevant category page for context.

---

## Last Complete Month

```sql eval_month_label
-- Identify the last complete month for display
-- Source: gold.signal_velocity_trends
-- Offset 1 skips the current partial month
SELECT DISTINCT record_month AS eval_month
FROM gold_signal_velocity_trends
ORDER BY record_month DESC
LIMIT 1 OFFSET 1
```

{% row %}
    {% big_value
        data="gold_signal_velocity_trends"
        value="sum(current_month_count)"
        title="Instruments"
        fmt="num0"
        where="category != 'Metadata'"
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
        fmt="usd0"
        where="category != 'Metadata'"
        date_range={
            date="record_month"
            range="previous month"
        }
        comparison={
            compare_vs="prior period"
            pct_fmt="pct0"
        }
    /%}
{% /row %}

> **Why this matters:**  
- These numbers reflect the most recent complete month, not the current partial month. The comparison arrow shows the direction and magnitude of change versus the month before.  
- Sharp drop in dollar volume with stable instrument count? Smaller transactions replacing larger ones.  
- Spike in instrument count with flat dollar volume? Administrative filings, not economic activity.  
- Green arrow = activity expanding. Red arrow = activity contracting. Check the category breakdown below for what moved.

---

## Month-over-Month by Subcategory

Sorted by largest absolute change in the most recent complete month. Signals where to look deeper.

```sql mom_by_subcategory
-- Month-over-month volume change per subcategory, last complete month
-- Source: gold.signal_velocity_trends
-- Subcategory values are dim-driven (source: dim.signal_category INSERTs in 01_dimensions)
-- Uses second-most-recent month (offset 1) to avoid partial-month distortion
-- pct_change divided by 100 for Evidence pct1 format rendering
-- Sorted by absolute change descending to surface biggest movers
-- pct1 retained on this table: subcategory-level precision aids triage
WITH eval AS (
    SELECT DISTINCT record_month
    FROM gold_signal_velocity_trends
    ORDER BY record_month DESC
    LIMIT 1 OFFSET 1
)
SELECT
    category,
    subcategory,
    CASE subcategory
        WHEN 'Equity Unlock Signals' THEN 'Releases, satisfactions, reconveyances'
        WHEN 'Owner Pressure Signals' THEN 'Tax liens, delinquencies, lis pendens'
        WHEN 'Forced Sale Signals' THEN 'Foreclosure deeds, tax deeds, auctions'
        WHEN 'Blocker Clearance Signals' THEN 'Cancellations, terminations, withdrawals'
        WHEN 'Transaction Timing Triggers' THEN 'Deeds, assignments, sales'
        WHEN 'Life and Legal Triggers' THEN 'Probate, death certificates, divorce'
        WHEN 'Deal Propensity Events' THEN 'Options, purchase contracts'
        WHEN 'Capital & Leverage Intelligence' THEN 'Mortgages, modifications, subordinations'
        WHEN 'Title and Ownership Complexity' THEN 'Quitclaims, corrections, affidavits'
        WHEN 'Deal Complexity Events' THEN 'Amendments, addenda, extensions'
        WHEN 'Supply Signals' THEN 'Construction permits, plats, condo declarations'
        WHEN 'Regulatory, Risk & Claims Intelligence' THEN 'Government liens, regulatory actions'
        WHEN 'Liens and Encumbrances' THEN 'Mechanic''s liens, judgment liens, tax liens'
        ELSE subcategory
    END AS what_it_tracks,
    current_month_count,
    prior_month_count,
    current_month_count - prior_month_count AS volume_change,
    ROUND(mom_pct_change / 100.0, 3) AS pct_change
FROM gold_signal_velocity_trends
WHERE category != 'Metadata'
  AND record_month = (SELECT record_month FROM eval)
ORDER BY ABS(current_month_count - prior_month_count) DESC
```

{% table data="mom_by_subcategory" page_size=200 %}
    {% dimension value="category" /%}
    {% dimension value="subcategory" /%}
    {% dimension value="what_it_tracks" title="What It Tracks" /%}
    {% measure value="current_month_count" title="Current" fmt="num0" /%}
    {% measure value="prior_month_count" title="Prior" fmt="num0" /%}
    {% measure value="volume_change" title="Change" fmt="num0" /%}
    {% measure value="pct_change" title="% Change" fmt="pct1" /%}
{% /table %}

> **So what?**  
- The top row had the biggest absolute volume swing.  
- Is that swing large relative to its historical average, or is this subcategory always volatile?  
- Cross-check: if it also appears in the Attention Now table above, the swing is statistically unusual. If not, it is within normal variation.  
- The "what it tracks" column tells you what documents are behind the number.  
- These two tables together separate noise from signal.

---

## Divergence Monitor

Two cross-category pairs that reveal structural shifts when they move in opposite directions.

```sql divergence_equity_pressure
-- Paired monthly trends: Equity Unlock Signals vs Owner Pressure Signals
-- Source: gold.signal_velocity_trends
-- Subcategory literals verified: 01_dimensions line 250, line 276
-- Purpose: opposite movement between these two signals structural borrower health shifts
-- Leading partial month trimmed: WHERE > MIN(record_month) avoids near-zero start
--   caused by silver.monthly_signal_velocity's 3-year CURRENT_DATE window
SELECT
    record_month,
    subcategory AS series_name,
    current_month_count AS monthly_count
FROM gold_signal_velocity_trends
WHERE subcategory IN ('Equity Unlock Signals', 'Owner Pressure Signals')
  AND record_month > (
      SELECT MIN(record_month)
      FROM gold_signal_velocity_trends
  )
ORDER BY record_month, subcategory
```

{% line_chart
    data="divergence_equity_pressure"
    x="record_month"
    y="monthly_count"
    series="series_name"
    title="Equity Unlocks vs Owner Pressure"
    y_fmt="num0"
    chart_options={
        series_colors={
            "Equity Unlock Signals"="#22c55e"
            "Owner Pressure Signals"="#f97316"
        }
    }
/%}

> **Why this matters:**  
- Green = Equity Unlock Signals (releases, satisfactions, reconveyances that free collateral).  
- Orange = Owner Pressure Signals (tax liens, delinquencies, lis pendens, foreclosure notices).  
- Equity unlocks rising while pressure falls = borrower health improving.  
- Lines crossing or pressure rising while unlocks fall = conditions tightening.  
- The relationship between the two lines matters more than either line alone.

```sql divergence_capital_forced
-- Paired monthly trends: Capital & Leverage vs Forced Sale Signals
-- Source: gold.signal_velocity_trends
-- Subcategory literals verified: 01_dimensions line 174, line 264
-- Purpose: capital activity declining while forced sales rise suggests tightening market
-- Leading partial month trimmed: same rationale as equity/pressure pair
SELECT
    record_month,
    subcategory AS series_name,
    current_month_count AS monthly_count
FROM gold_signal_velocity_trends
WHERE subcategory IN ('Capital & Leverage Intelligence', 'Forced Sale Signals')
  AND record_month > (
      SELECT MIN(record_month)
      FROM gold_signal_velocity_trends
  )
ORDER BY record_month, subcategory
```

{% line_chart
    data="divergence_capital_forced"
    x="record_month"
    y="monthly_count"
    series="series_name"
    title="Capital and Leverage vs Forced Sales"
    y_fmt="num0"
    chart_options={
        series_colors={
            "Capital & Leverage Intelligence"="#3b82f6"
            "Forced Sale Signals"="#f97316"
        }
    }
/%}

> **Why this matters:**  
- Blue = Capital and Leverage Intelligence (mortgages, modifications, subordinations, bonds, promissory notes).  
- Orange = Forced Sale Signals (foreclosure deeds, tax deeds, auction confirmations).  
- Rising capital activity with stable or declining forced sales = active lending market, manageable distress.  
- Declining lending while forced sales increase = market tightening, creating both counseling needs and discounted acquisition opportunities.

---

## Market Pulse

Category-level summary for the most recent complete month. Each category answers a different lending question.

```sql market_pulse
-- Category-level summary with five-questions framework and 6-month average
-- Source: gold.signal_velocity_trends, last complete month (offset 1)
-- Category literals verified: 01_dimensions lines 174, 202, 212, 224, 335
-- Question mappings from dashboard architecture plan v2, lines 43-49
-- Excludes Metadata (administrative filings with no signal value)
-- 6-month average: trailing 6 complete months (offset 1 through 6), per architecture plan spec
WITH eval AS (
    SELECT DISTINCT record_month
    FROM gold_signal_velocity_trends
    ORDER BY record_month DESC
    LIMIT 1 OFFSET 1
),
six_months AS (
    SELECT DISTINCT record_month
    FROM gold_signal_velocity_trends
    ORDER BY record_month DESC
    LIMIT 6 OFFSET 1
)
SELECT
    category,
    CASE category
        WHEN 'Transaction Propensity Intelligence' THEN 'Who will transact next?'
        WHEN 'Transaction Feasibility & Close Intelligence' THEN 'Will this deal close?'
        WHEN 'Capital & Leverage Intelligence' THEN 'What is the capital position?'
        WHEN 'Market Dynamics Intelligence' THEN 'Where is the market moving?'
        WHEN 'Regulatory, Risk & Claims Intelligence' THEN 'Where are the compliance risks?'
    END AS question,
    SUM(CASE WHEN record_month = (SELECT record_month FROM eval)
        THEN current_month_count ELSE 0 END) AS current_count,
    SUM(CASE WHEN record_month = (SELECT record_month FROM eval)
        THEN prior_month_count ELSE 0 END) AS prior_count,
    SUM(CASE WHEN record_month = (SELECT record_month FROM eval)
        THEN current_month_count ELSE 0 END)
    - SUM(CASE WHEN record_month = (SELECT record_month FROM eval)
        THEN prior_month_count ELSE 0 END) AS delta,
    ROUND(
        SUM(CASE WHEN record_month IN (SELECT record_month FROM six_months)
            THEN current_month_count ELSE 0 END) / 6.0,
        0
    ) AS avg_6mo
FROM gold_signal_velocity_trends
WHERE category != 'Metadata'
  AND (record_month = (SELECT record_month FROM eval)
    OR record_month IN (SELECT record_month FROM six_months))
GROUP BY category
ORDER BY SUM(CASE WHEN record_month = (SELECT record_month FROM eval)
    THEN current_month_count ELSE 0 END) DESC
```

{% table data="market_pulse" page_size=200 %}
    {% dimension value="category" /%}
    {% dimension value="question" /%}
    {% measure value="current_count" title="Current" fmt="num0" /%}
    {% measure value="prior_count" title="Prior" fmt="num0" /%}
    {% measure value="delta" title="Delta" fmt="num0" /%}
    {% measure value="avg_6mo" title="6-Mo Avg" fmt="num0" /%}
{% /table %}

> **So what?**  
- Each row is a different dimension of the lending market. The delta column shows expansion or contraction relative to the month before. The 6-month average provides a baseline so you can judge whether the current month is unusually high, low, or typical.  
- Current count well above the 6-month average? Acceleration. Well below? Deceleration.  
- Do all five tell a coherent story, or are some moving in unexpected directions?  
- The category pages provide subcategory-level detail behind each row.

---

## YTD Pace vs Prior Year

Apples-to-apples comparison of year-to-date instrument volume across recent years.

```sql ytd_pace
-- Year-over-year instrument volume and dollar totals
-- Source: gold.yearly_instrument_summary
-- Excludes Metadata category
-- Note: current year is partial (check data quality section for most recent recording date)
SELECT
    recorded_year,
    SUM(instrument_count) AS total_instruments,
    ROUND(SUM(total_amount) / 1000000.0, 1) AS dollars_millions
FROM gold_yearly_instrument_summary
WHERE category != 'Metadata'
  AND recorded_year >= 2022
GROUP BY recorded_year
ORDER BY recorded_year
```

{% bar_chart
    data="ytd_pace"
    x="recorded_year"
    y="total_instruments"
    title="Annual Instrument Volume"
    y_fmt="num0"
    x_fmt="id"
/%}

{% table data="ytd_pace" page_size=200 %}
    {% dimension value="recorded_year" title="Year" /%}
    {% measure value="total_instruments" title="Total Instruments" fmt="num0" /%}
    {% measure value="dollars_millions" title="Dollars (M)" fmt="num1" /%}
{% /table %}

> **So what?**  
- The most recent year is partial. Compare it to the same calendar window in prior years before drawing conclusions.  
- Is total volume trending up, down, or flat across the full years?  
- Rising instrument count with falling dollar volume = shift toward smaller transactions.  
- Declining count with rising dollars = fewer but larger deals.

---

## Signal Concentration

Which subcategories drive the majority of recorded activity? Pareto analysis showing cumulative share of total volume and dollar value.

```sql signal_concentration
-- Pareto analysis: subcategory share of total volume and dollars, trailing 12 months
-- Source: gold.signal_velocity_trends
-- Window: trailing 12 months (12 most recent record_month values)
-- Excludes Metadata
-- Percentages as 0-1 decimals for Evidence pct1 format
-- Measure-only table with order attribute preserves both fmt formatting and sort order
WITH trailing_twelve AS (
    SELECT DISTINCT record_month
    FROM gold_signal_velocity_trends
    ORDER BY record_month DESC
    LIMIT 12
),

subcategory_totals AS (
    SELECT
        subcategory,
        SUM(current_month_count) AS volume,
        ROUND(SUM(current_month_amount), 0) AS dollars
    FROM gold_signal_velocity_trends
    WHERE category != 'Metadata'
      AND record_month IN (SELECT record_month FROM trailing_twelve)
    GROUP BY subcategory
),

with_cumulative AS (
    SELECT
        subcategory,
        volume,
        dollars,
        volume * 1.0 / SUM(volume) OVER () AS vol_pct,
        SUM(volume) OVER (ORDER BY volume DESC) * 1.0 / SUM(volume) OVER () AS vol_cumulative,
        dollars * 1.0 / NULLIF(SUM(dollars) OVER (), 0) AS dollar_pct,
        SUM(dollars) OVER (ORDER BY dollars DESC) * 1.0 / NULLIF(SUM(dollars) OVER (), 0) AS dollar_cumulative
    FROM subcategory_totals
)

SELECT
    subcategory,
    volume,
    vol_pct,
    vol_cumulative,
    dollars,
    dollar_pct,
    dollar_cumulative
FROM with_cumulative
ORDER BY volume DESC
```

{% table data="signal_concentration" page_size=200 order="volume desc" %}
    {% measure value="subcategory" title="Subcategory" /%}
    {% measure value="volume" title="Volume" fmt="num0" /%}
    {% measure value="vol_pct" title="% of Total" fmt="pct1" /%}
    {% measure value="vol_cumulative" title="Cumulative %" fmt="pct1" /%}
    {% measure value="dollars" title="Dollars" fmt="usd0" /%}
    {% measure value="dollar_pct" title="$ % of Total" fmt="pct1" /%}
    {% measure value="dollar_cumulative" title="$ Cumulative %" fmt="pct1" /%}
{% /table %}

> **So what?**  
- Subcategories above the 80% cumulative threshold are the volume drivers. Changes in those subcategories move the whole market.  
- Compare the volume Pareto to the dollar Pareto. A subcategory that is low-volume but high-dollar (like Capital and Leverage) carries outsized financial significance per filing.  
- Subcategories below 80% in volume may carry high individual significance: a single forced sale or deal propensity event can represent more dollar opportunity than dozens of routine filings.  
- Volume concentration tells you where to look for trends. Dollar concentration tells you where the money is.

---

## Annual Trend

Year-over-year volume by signal category. Reveals structural shifts in the composition of recorded activity.

```sql annual_trend
-- Annual volume by category, pivoted by year
-- Source: gold.yearly_instrument_summary
-- Excludes Metadata
-- recorded_year cast to date for Evidence pivot date_grain="year"
-- Recent years including current partial year
SELECT
    category,
    DATE(recorded_year || '-01-01') AS record_date,
    SUM(instrument_count) AS volume
FROM gold_yearly_instrument_summary
WHERE category != 'Metadata'
  AND recorded_year >= 2022
GROUP BY category, recorded_year
ORDER BY category, recorded_year
```

{% table data="annual_trend" %}
    {% dimension value="category" /%}
    {% pivot value="record_date" date_grain="year" /%}
    {% measure value="sum(volume)" fmt="num0" /%}
{% /table %}

> **So what?**  
- Are the proportions between categories stable year over year, or is one growing at the expense of others?  
- Transaction Propensity shrinking while Capital and Leverage grows = market transitioning from high transaction activity to a refinance-heavy environment.  
- Does the most recent partial year's category mix look like prior years scaled down, or is the composition itself changing?