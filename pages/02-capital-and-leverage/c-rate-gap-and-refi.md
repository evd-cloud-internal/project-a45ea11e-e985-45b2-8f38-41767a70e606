---
name: C Rate Gap & Refi
assetId: e2f53792-aba3-459a-877c-21b567d09b60
type: page
---

---
title: Rate Gap & Refi
---

# Which mortgages have a rate-driven opportunity?

Capital and Leverage Intelligence asks: "What is the capital position?" This page answers the operational follow-up: given the current rate environment, how many active mortgages are overpaying relative to today's rates, and who should a loan officer call first? Rate gap tiers organize every mortgage into an actionable classification, from strong refinance candidates to retention targets worth protecting.

```sql portfolio_all_types
-- Portfolio-level rate gap sizing (ALL_TYPES rollup)
-- gold.rate_gap_portfolio_summary filtered to tier-only aggregation
SELECT
    CASE rate_gap_opportunity
        WHEN 'STRONG_REFI' THEN '1 - Strong Refi'
        WHEN 'MILD_REFI' THEN '2 - Mild Refi'
        WHEN 'NEUTRAL' THEN '3 - Neutral'
        WHEN 'RETENTION' THEN '4 - Retention'
        WHEN 'STRONG_RETENTION' THEN '5 - Strong Retention'
    END AS tier
  , active_count
  , active_principal
  , wtd_avg_origination_rate
  , wtd_avg_rate_gap
  , refi_sweet_spot_count
  , govt_streamline_count
  , pct_of_active_principal
FROM gold_rate_gap_portfolio_summary
WHERE detail_level = 'ALL_TYPES'
ORDER BY tier
```

## Portfolio Sizing

```sql portfolio_kpis
-- Top-line KPIs from the rate gap portfolio
SELECT
    sumIf(active_count, tier LIKE '1%' OR tier LIKE '2%') AS refi_eligible_count
  , sumIf(active_principal, tier LIKE '1%' OR tier LIKE '2%') AS refi_eligible_principal
  , sumIf(active_count, tier LIKE '4%' OR tier LIKE '5%') AS retention_count
  , sumIf(active_principal, tier LIKE '4%' OR tier LIKE '5%') AS retention_principal
FROM {{portfolio_all_types}} AS portfolio_all_types
```

{% big_value
    data="portfolio_kpis"
    value="refi_eligible_count"
    title="Refi-Eligible Mortgages"
    fmt="num0"
/%}

{% big_value
    data="portfolio_kpis"
    value="refi_eligible_principal"
    title="Refi-Eligible Principal"
    fmt="usd0m"
/%}

{% big_value
    data="portfolio_kpis"
    value="retention_count"
    title="Retention Targets"
    fmt="num0"
/%}

{% big_value
    data="portfolio_kpis"
    value="retention_principal"
    title="Retention Principal"
    fmt="usd0m"
/%}

> Refi-eligible includes Strong Refi and Mild Refi tiers (rate gap of +0.50% or more). Retention includes Retention and Strong Retention tiers (rate gap of -0.50% or less). Neutral tier mortgages are neither.

{% table data="portfolio_all_types" %}
    {% dimension value="tier" title="Rate Gap Tier" /%}
    {% measure value="active_count" title="Active Mortgages" fmt="num0" /%}
    {% measure value="active_principal" title="Active Principal" fmt="usd0m" /%}
    {% measure value="pct_of_active_principal" title="% of Portfolio" fmt="num1" /%}
    {% measure value="wtd_avg_origination_rate" title="Avg Orig Rate" fmt="num2" /%}
    {% measure value="wtd_avg_rate_gap" title="Avg Rate Gap" fmt="num2" /%}
    {% measure value="govt_streamline_count" title="Govt Streamline" fmt="num0" /%}
{% /table %}

## Rate Gap by Product Type

```sql portfolio_by_type
-- Rate gap sizing broken out by mortgage product type
SELECT
    CASE rate_gap_opportunity
        WHEN 'STRONG_REFI' THEN '1 - Strong Refi'
        WHEN 'MILD_REFI' THEN '2 - Mild Refi'
        WHEN 'NEUTRAL' THEN '3 - Neutral'
        WHEN 'RETENTION' THEN '4 - Retention'
        WHEN 'STRONG_RETENTION' THEN '5 - Strong Retention'
    END AS tier
  , mortgage_type_group AS product
  , active_count
  , active_principal
  , wtd_avg_rate_gap
  , govt_streamline_count
FROM gold_rate_gap_portfolio_summary
WHERE detail_level = 'BY_TYPE'
ORDER BY
    CASE rate_gap_opportunity
        WHEN 'STRONG_REFI' THEN 1
        WHEN 'MILD_REFI' THEN 2
        WHEN 'NEUTRAL' THEN 3
        WHEN 'RETENTION' THEN 4
        WHEN 'STRONG_RETENTION' THEN 5
    END
  , mortgage_type_group
```

{% table data="portfolio_by_type" page_size=200 %}
    {% dimension value="tier" title="Tier" /%}
    {% dimension value="product" title="Product" /%}
    {% measure value="active_count" title="Active" fmt="num0" /%}
    {% measure value="active_principal" title="Principal" fmt="usd0m" /%}
    {% measure value="wtd_avg_rate_gap" title="Avg Gap" fmt="num2" /%}
    {% measure value="govt_streamline_count" title="Govt Streamline" fmt="num0" /%}
{% /table %}

The product breakdown reveals where the refi opportunity concentrates. Government-backed loans (VA, FHA) with positive rate gaps qualify for streamlined refinance programs that require no appraisal and minimal paperwork. Consumer conventional mortgages need a full underwrite but often represent the largest dollar opportunity per loan.

> **Rate gap tier definitions.** Strong Refi: origination rate is 1.50+ points above current Guam effective rate (clear savings, call now). Mild Refi: 0.50 to 1.49 points above (savings depend on closing costs). Neutral: within 0.50 points either direction (no action). Retention: 0.50 to 1.49 points below current rate (borrower has a good deal; protect it). Strong Retention: 1.50+ points below (borrower has a significantly cheap loan; high flight risk if approached by competitors).

## When were today's refi targets originated?

```sql rate_history
-- Full rate history from dim.interest_rates
-- national_mortgage_30yr starts April 1971; guam_effective_rate derived
SELECT
    rate_month
  , national_mortgage_30yr
  , guam_effective_rate
FROM dim_interest_rates
WHERE national_mortgage_30yr IS NOT NULL
ORDER BY rate_month
```

```sql current_rate_ref
-- Current Guam effective rate for reference line
-- Borrowers above this line are refi candidates; below are retention targets
SELECT
    guam_effective_rate AS current_rate
  , 'Current Guam Rate' AS rate_label
FROM dim_interest_rates
WHERE guam_effective_rate IS NOT NULL
ORDER BY rate_month DESC NULLS LAST
LIMIT 1
```

{% line_chart
    data="rate_history"
    x="rate_month"
    y=["national_mortgage_30yr", "guam_effective_rate"]
    y_fmt="num1"
    title="National 30-Year Mortgage and Guam Effective Rate (1971 to Present)"
    subtitle="Mortgages originated above the reference line are refi candidates at today's rate"
    chart_options={
        series_colors={
            "national_mortgage_30yr"="#006BA4"
            "guam_effective_rate"="#FF800E"
        }
    }
    y_axis_options={
        fit_to_data=true
    }
%}
    {% reference_line
        data="current_rate_ref"
        y="current_rate"
        label="rate_label"
        color="#22c55e"
        line_options={ type="dashed" width=2 }
        label_options={ position="above_end" }
    /%}
    {% reference_area
        x_min="2022-01-01"
        x_max="2024-06-01"
        label="Rate Shock Vintage"
        color="#ef4444"
        area_options={ opacity=0.08 }
    /%}
{% /line_chart %}

The green dashed line marks today's Guam effective rate. Every mortgage originated when rates were above that line carries a positive rate gap; the borrower is paying more than they would on a new loan today. The red-shaded band highlights the 2022 to 2024 rate shock vintage, where origination rates peaked and the largest concentration of active refi-eligible mortgages sits.

Guam's effective rate tracks the national rate with a small negative spread (currently about -0.33%, derived from a single lender snapshot). Potential lending partner data will replace this with actual local pricing.

> The refi sweet spot flag (is_refi_sweet_spot) marks mortgages 24 to 60 months old, which is the traditional outreach window based on seasoning. This flag is a secondary filter within the work queues below for loan officers who want to layer timing on top of rate qualification. Rate gap tiers are the primary organizing concept.

---

## Work Queues

Eight contact-ready queues, one per product type or eligibility criterion. Each queue is sorted by dollar opportunity or priority. Contact data coverage is incomplete in the source records; the potential lending partner will supplement via skip tracing.

### Rate Gap Queue (All Products)

The primary work queue. Every active mortgage with a known rate gap, sorted by dollar opportunity (principal times rate gap). Refi targets sort above retention targets naturally because positive gaps produce larger dollar values.

```sql rate_gap_queue
-- Primary rate gap work queue: all products, sorted by dollar opportunity
-- report.rate_gap_work_queue
SELECT
    instrument_number
  , mortgage_type_group AS product
  , principal
  , rate_gap_opportunity AS tier
  , rate_gap_guam AS rate_gap
  , rate_gap_dollar_opportunity AS dollar_opportunity
  , vintage_year
  , contact_name
  , contact_phone
FROM report_rate_gap_work_queue
ORDER BY dollar_opportunity DESC NULLS LAST
```

{% table data="rate_gap_queue" page_size=25 %}
    {% dimension value="instrument_number" title="Instrument" /%}
    {% dimension value="product" title="Product" /%}
    {% measure value="principal" title="Principal" fmt="usd0" /%}
    {% dimension value="tier" title="Tier" /%}
    {% measure value="rate_gap" title="Gap %" fmt="num2" /%}
    {% measure value="dollar_opportunity" title="$ Opportunity" fmt="usd0" /%}
    {% dimension value="vintage_year" title="Vintage" /%}
    {% dimension value="contact_name" title="Contact" /%}
    {% dimension value="contact_phone" title="Phone" /%}
{% /table %}

### VA IRRRL Opportunities

VA loans eligible for the Interest Rate Reduction Refinance Loan. No appraisal, no income verification. Minimum 6 months from origination. The easiest refi product available.

```sql va_irrrl_queue
-- VA IRRRL eligible mortgages
-- report.va_irrrl_opportunities
SELECT
    instrument_number
  , principal
  , months_since_executed AS months
  , vintage_year
  , value_tier
  , name_full AS contact
  , phone_number AS phone
FROM report_va_irrrl_opportunities
ORDER BY principal DESC NULLS LAST
```

{% table data="va_irrrl_queue" page_size=25 %}
    {% dimension value="instrument_number" title="Instrument" /%}
    {% measure value="principal" title="Principal" fmt="usd0" /%}
    {% measure value="months" title="Months" fmt="num0" /%}
    {% dimension value="vintage_year" title="Vintage" /%}
    {% dimension value="value_tier" title="Value Tier" /%}
    {% dimension value="contact" title="Contact" /%}
    {% dimension value="phone" title="Phone" /%}
{% /table %}

### FHA Streamline Opportunities

FHA loans eligible for streamline refinance. Minimum 7 months from origination. Similar to VA IRRRL: reduced documentation, no appraisal required in most cases.

```sql fha_streamline_queue
-- FHA streamline eligible mortgages
-- report.fha_streamline_opportunities
SELECT
    instrument_number
  , principal
  , months_since_executed AS months
  , vintage_year
  , value_tier
  , name_full AS contact
  , phone_number AS phone
FROM report_fha_streamline_opportunities
ORDER BY principal DESC NULLS LAST
```

{% table data="fha_streamline_queue" page_size=25 %}
    {% dimension value="instrument_number" title="Instrument" /%}
    {% measure value="principal" title="Principal" fmt="usd0" /%}
    {% measure value="months" title="Months" fmt="num0" /%}
    {% dimension value="vintage_year" title="Vintage" /%}
    {% dimension value="value_tier" title="Value Tier" /%}
    {% dimension value="contact" title="Contact" /%}
    {% dimension value="phone" title="Phone" /%}
{% /table %}

### HELOC Reset Outreach

HELOCs approaching the end of their draw period (8 to 10 years from origination). When the draw period ends, the borrower's payment structure changes; if they don't convert to a fixed product, they face payment shock. Proactive outreach converts a potential crisis into a new lending relationship.

```sql heloc_reset_queue
-- HELOCs approaching draw period end
-- report.heloc_reset_outreach
SELECT
    instrument_number
  , credit_line_amount AS credit_line
  , years_since_executed AS years
  , reset_status
  , name_full AS contact
  , phone_number AS phone
FROM report_heloc_reset_outreach
ORDER BY years DESC NULLS LAST, credit_line DESC NULLS LAST
```

{% table data="heloc_reset_queue" page_size=25 %}
    {% dimension value="instrument_number" title="Instrument" /%}
    {% measure value="credit_line" title="Credit Line" fmt="usd0" /%}
    {% measure value="years" title="Years" fmt="num1" /%}
    {% dimension value="reset_status" title="Status" /%}
    {% dimension value="contact" title="Contact" /%}
    {% dimension value="phone" title="Phone" /%}
{% /table %}

### Construction Conversion

Construction loans ready for permanent financing. Construction loans have short terms (typically 18 months). Loans past 6 months are entering the conversion window; past 24 months they are overdue for permanent financing.

```sql construction_queue
-- Construction loans by conversion readiness
-- report.construction_followup
SELECT
    instrument_number
  , amount
  , months_since_executed AS months
  , conversion_status
  , name_full AS contact
  , phone_number AS phone
FROM report_construction_followup
ORDER BY months_since_executed DESC NULLS LAST
```

{% table data="construction_queue" page_size=25 %}
    {% dimension value="instrument_number" title="Instrument" /%}
    {% measure value="amount" title="Amount" fmt="usd0" /%}
    {% measure value="months" title="Months" fmt="num0" /%}
    {% dimension value="conversion_status" title="Status" /%}
    {% dimension value="contact" title="Contact" /%}
    {% dimension value="phone" title="Phone" /%}
{% /table %}

### Private Mortgage Conversion

Private (hard money) mortgages that are 12+ months seasoned. Private mortgages typically carry rates of 10 to 15% or higher. Converting these borrowers to a conventional bank product saves them significant money and creates a new banking relationship.

```sql private_queue
-- Private mortgages ready for bank conversion
-- report.private_mortgage_conversion
SELECT
    instrument_number
  , principal
  , months_since_executed AS months
  , conversion_readiness
  , name_full AS contact
  , phone_number AS phone
FROM report_private_mortgage_conversion
ORDER BY principal DESC NULLS LAST
```

{% table data="private_queue" page_size=25 %}
    {% dimension value="instrument_number" title="Instrument" /%}
    {% measure value="principal" title="Principal" fmt="usd0" /%}
    {% measure value="months" title="Months" fmt="num0" /%}
    {% dimension value="conversion_readiness" title="Readiness" /%}
    {% dimension value="contact" title="Contact" /%}
    {% dimension value="phone" title="Phone" /%}
{% /table %}

### Retention Alerts

Recent equity unlock signals (satisfactions, releases, reconveyances) on existing mortgages. A satisfied mortgage means the borrower either refinanced with a competitor or paid off the loan entirely. Both outcomes represent either a lost relationship or a HELOC opportunity. Time-sensitivity is high: the first 7 days after recording are the best outreach window.

```sql retention_queue
-- Recent equity unlocks for retention outreach
-- report.lender_retention_alerts
SELECT
    instrument_number
  , amount
  , equity_unlock_type
  , days_since_recorded AS days
  , priority_tier
  , name_full AS contact
  , phone_number AS phone
FROM report_lender_retention_alerts
ORDER BY days ASC NULLS LAST
```

{% table data="retention_queue" page_size=25 %}
    {% dimension value="instrument_number" title="Instrument" /%}
    {% measure value="amount" title="Amount" fmt="usd0" /%}
    {% dimension value="equity_unlock_type" title="Type" /%}
    {% measure value="days" title="Days" fmt="num0" /%}
    {% dimension value="priority_tier" title="Priority" /%}
    {% dimension value="contact" title="Contact" /%}
    {% dimension value="phone" title="Phone" /%}
{% /table %}

### Refi Outreach List

Mortgages in the traditional refi sweet spot (24 to 60 months from origination), regardless of rate gap tier. This is the secondary queue: loan officers who want to layer seasoning timing on top of rate qualification use this list. The primary rate gap queue above captures the rate-driven opportunity; this queue captures the timing-driven opportunity.

```sql refi_outreach_queue
-- Refi sweet spot outreach list (24-60 month window)
-- report.refi_outreach_list
SELECT
    instrument_number
  , mortgage_type_group AS product
  , principal
  , months_since_executed AS months
  , value_tier
  , name_full AS contact
  , phone_number AS phone
FROM report_refi_outreach_list
ORDER BY principal DESC NULLS LAST
```

{% table data="refi_outreach_queue" page_size=25 %}
    {% dimension value="instrument_number" title="Instrument" /%}
    {% dimension value="product" title="Product" /%}
    {% measure value="principal" title="Principal" fmt="usd0" /%}
    {% measure value="months" title="Months" fmt="num0" /%}
    {% dimension value="value_tier" title="Value Tier" /%}
    {% dimension value="contact" title="Contact" /%}
    {% dimension value="phone" title="Phone" /%}
{% /table %}

---

## Mortgage Product Reference

```sql mortgage_products
-- dim.mortgage_type: 14 rows mapping trusted types to product groups
SELECT
    mortgage_type AS trusted_type
  , mortgage_type_group AS product_group
FROM dim_mortgage_type
ORDER BY mortgage_type_group, mortgage_type
```

```sql term_assumptions
-- dim.mortgage_term_assumptions: assumed lifecycle by product
SELECT
    mortgage_trusted_type AS trusted_type
  , assumed_term_label AS term
  , assumed_term_months AS months
FROM dim_mortgage_term_assumptions
ORDER BY assumed_term_months DESC NULLS LAST
```

{% row %}
{% table data="mortgage_products" page_size=200 %}
    {% dimension value="trusted_type" title="Mortgage Type" /%}
    {% dimension value="product_group" title="Product Group" /%}
{% /table %}

{% table data="term_assumptions" page_size=200 %}
    {% dimension value="trusted_type" title="Mortgage Type" /%}
    {% dimension value="term" title="Term" /%}
    {% measure value="months" title="Months" fmt="num0" /%}
{% /table %}
{% /row %}

> Product groups determine which work queue a mortgage appears in. Term assumptions determine the is_likely_active flag: a mortgage is considered active if fewer months have elapsed since execution than the assumed term. These assumptions are conservative defaults; actual loan terms vary.

## So what?

How large is the refi-eligible population under current rates, and which product types concentrate the opportunity? The Rate Environment page shows where rates could go next; the tier distribution on this page shows what that means in dollars and mortgages. A loan officer starts with the primary rate gap queue (sorted by dollar opportunity), then checks the specialty queues for VA IRRRL, FHA streamline, and HELOC reset candidates that qualify for simplified processing.