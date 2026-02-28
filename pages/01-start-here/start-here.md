---
name: Beacons
assetId: 7a2d6c4f-0f68-4ed7-b360-85d5ac5f5013
type: page
---

---
Beacons
---

<!-- ================================================================
     Beacons: Landing Page (Page 1)
     Source views: gold_signal_velocity_trends,
                   audit_signal_category_coverage,
                   audit_bridge_relationship_coverage,
                   audit_mortgage_eligibility_flags,
                   audit_data_freshness
     Page type: Executive (whole-number formatting: pct0, num0, usd0m)
     Design: Orientation and wayfinding. First-time visitors get
             oriented; returning users navigate via sidebar.
     v9: Redesigned from v8 (11 sections) to landing page (3 sections).
         Moved to category pages: Attention Now, Divergence Monitor,
         MoM by Subcategory, Last Complete Month.
         Moved to appendix: Signal Families, Signal Concentration,
         YTD Pace, Annual Trend.
     ================================================================ -->

# Beacons

Beacons transforms Guam Department of Land Management recorded instruments into predictive lending intelligence. Every instrument is classified into a signal family, scored on three axes (strength, revenue potential, distress severity), and organized into prioritized work queues for lending teams.

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

> These numbers define the platform's analytical reach. Classification coverage determines what percentage of the recorded universe produces signals. Property linkage determines how many instruments can be traced to a physical asset. Active mortgages define the addressable refinance market. If days since last record exceeds 7, investigate the pipeline before trusting the analytical pages.

---

## Five Questions

Every signal in the platform maps to one of five lending questions. Each question has its own section in the sidebar with full analysis, control charts, and work queues.

```sql market_pulse
-- Category-level summary with five-questions framework and 6-month average
-- Source: gold.signal_velocity_trends, last complete month (offset 1)
-- Category literals verified: 01_dimensions lines 174, 202, 212, 224, 335
-- Question mappings from dashboard architecture plan v2, lines 43-49
-- Excludes Metadata (administrative filings with no signal value)
-- 6-month average: trailing 6 complete months (offset 1 through 6)
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
    CASE category
        WHEN 'Transaction Propensity Intelligence' THEN 'Who will transact next?'
        WHEN 'Transaction Feasibility & Close Intelligence' THEN 'Will this deal close?'
        WHEN 'Capital & Leverage Intelligence' THEN 'What is the capital position?'
        WHEN 'Market Dynamics Intelligence' THEN 'Where is the market moving?'
        WHEN 'Regulatory, Risk & Claims Intelligence' THEN 'Where are the compliance risks?'
    END AS question,
    CASE category
        WHEN 'Transaction Propensity Intelligence' THEN 'Market Activity'
        WHEN 'Transaction Feasibility & Close Intelligence' THEN 'Deal Readiness'
        WHEN 'Capital & Leverage Intelligence' THEN 'Capital & Leverage'
        WHEN 'Market Dynamics Intelligence' THEN 'Market Dynamics'
        WHEN 'Regulatory, Risk & Claims Intelligence' THEN 'Regulatory & Risk'
    END AS sidebar_folder,
    SUM(CASE WHEN record_month = (SELECT record_month FROM eval)
        THEN current_month_count ELSE 0 END) AS current_month,
    SUM(CASE WHEN record_month = (SELECT record_month FROM eval)
        THEN current_month_count ELSE 0 END)
    - SUM(CASE WHEN record_month = (SELECT record_month FROM eval)
        THEN prior_month_count ELSE 0 END) AS change,
    ROUND(
        SUM(CASE WHEN record_month IN (SELECT record_month FROM six_months)
            THEN current_month_count ELSE 0 END) / 6.0,
        0
    ) AS six_month_avg
FROM gold_signal_velocity_trends
WHERE category != 'Metadata'
  AND (record_month = (SELECT record_month FROM eval)
    OR record_month IN (SELECT record_month FROM six_months))
GROUP BY category
ORDER BY
    CASE category
        WHEN 'Transaction Propensity Intelligence' THEN 1
        WHEN 'Transaction Feasibility & Close Intelligence' THEN 2
        WHEN 'Capital & Leverage Intelligence' THEN 3
        WHEN 'Market Dynamics Intelligence' THEN 4
        WHEN 'Regulatory, Risk & Claims Intelligence' THEN 5
    END
```

{% table data="market_pulse" page_size=200 %}
    {% dimension value="question" /%}
    {% dimension value="sidebar_folder" title="Sidebar" /%}
    {% measure value="current_month" title="Last Month" fmt="num0" /%}
    {% measure value="change" title="Change" fmt="num0" /%}
    {% measure value="six_month_avg" title="6-Mo Avg" fmt="num0" /%}
{% /table %}

> Each row is a different dimension of the lending market. Current month well above the 6-month average means acceleration; well below means deceleration. Click any folder in the sidebar to see subcategory breakdowns, control charts, and the work queues behind each number.