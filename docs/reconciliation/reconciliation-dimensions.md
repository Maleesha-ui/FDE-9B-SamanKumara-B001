# Reconciliation Dimensions

## Overview

Reconciliation ensures data integrity between source (SAP S/4HANA) and target (Zetheta FinSight) systems. This document defines the four reconciliation dimensions used to validate data completeness, accuracy, timeliness, and consistency across the integration pipeline.

---

## 1. Completeness

### Definition
All records from the source system are present in the target system without omissions. No data loss should occur during extraction, transformation, and loading.

### Formula
Completeness = (Target Records / Source Records) × 100

text

### Thresholds
| Category | Threshold | Status | Action |
|----------|-----------|--------|--------|
| **Pass** | ≥ 99.5% | ✅ PASS | Accept and proceed |
| **Warning** | 95% - 99.49% | ⚠️ WARNING | Investigate, continue with caution |
| **Fail** | < 95% | ❌ FAIL | Block/Pause, escalate immediately |

### Validation Method
```sql
-- Completeness check SQL
SELECT 
    COUNT(*) AS source_count,
    SUM(CASE WHEN target_id IS NOT NULL THEN 1 ELSE 0 END) AS matched_count,
    SUM(CASE WHEN target_id IS NULL THEN 1 ELSE 0 END) AS missing_count,
    (SUM(CASE WHEN target_id IS NOT NULL THEN 1 ELSE 0 END) / COUNT(*)) * 100 AS completeness_pct,
    CASE 
        WHEN (SUM(CASE WHEN target_id IS NOT NULL THEN 1 ELSE 0 END) / COUNT(*)) * 100 >= 99.5 
        THEN 'PASS'
        WHEN (SUM(CASE WHEN target_id IS NOT NULL THEN 1 ELSE 0 END) / COUNT(*)) * 100 >= 95 
        THEN 'WARNING'
        ELSE 'FAIL'
    END AS status
FROM source_records s
LEFT JOIN target_records t ON s.id = t.source_id
WHERE s.batch_id = 'BATCH-GL-20260328-0930';
Missing Records Investigation
sql
-- Identify missing records
SELECT 
    s.id,
    s.document_id,
    s.batch_id,
    s.extracted_at,
    'Missing in target' AS reason
FROM source_records s
LEFT JOIN target_records t ON s.id = t.source_id
WHERE t.source_id IS NULL
    AND s.batch_id = 'BATCH-GL-20260328-0930';
Completeness Tracking
Dimension	Daily Target	Weekly Target	Monthly Target
Completeness	≥ 99.5%	≥ 99.5%	≥ 99.8%
Records Checked	All batch records	All batch records	All batch records
Sample Size	100%	100%	100%
2. Accuracy
Definition
Data values in the target system correctly represent the source system values without transformation errors, rounding issues, or data corruption.

Formula
text
Accuracy = (Matching Field Values / Total Field Values) × 100
Thresholds
Category	Threshold	Status	Action
Pass	≥ 99.9%	✅ PASS	Accept and proceed
Warning	99% - 99.89%	⚠️ WARNING	Investigate specific fields
Fail	< 99%	❌ FAIL	Block/Pause, escalate immediately
Validation Method
sql
-- Accuracy check SQL
WITH field_comparison AS (
    SELECT 
        s.id,
        CASE WHEN s.document_id = t.document_id THEN 1 ELSE 0 END AS doc_match,
        CASE WHEN s.amount = t.amount THEN 1 ELSE 0 END AS amount_match,
        CASE WHEN s.currency = t.currency THEN 1 ELSE 0 END AS currency_match,
        CASE WHEN s.cost_centre = t.cost_centre THEN 1 ELSE 0 END AS cc_match,
        CASE WHEN s.posting_date = t.posting_date THEN 1 ELSE 0 END AS date_match
    FROM source_records s
    JOIN target_records t ON s.id = t.source_id
    WHERE s.batch_id = 'BATCH-GL-20260328-0930'
)
SELECT 
    COUNT(*) AS total_records,
    SUM(doc_match) AS doc_matches,
    SUM(amount_match) AS amount_matches,
    SUM(currency_match) AS currency_matches,
    SUM(cc_match) AS cc_matches,
    SUM(date_match) AS date_matches,
    (SUM(doc_match + amount_match + currency_match + cc_match + date_match) / (COUNT(*) * 5)) * 100 AS accuracy_pct,
    CASE 
        WHEN (SUM(doc_match + amount_match + currency_match + cc_match + date_match) / (COUNT(*) * 5)) * 100 >= 99.9 
        THEN 'PASS'
        WHEN (SUM(doc_match + amount_match + currency_match + cc_match + date_match) / (COUNT(*) * 5)) * 100 >= 99 
        THEN 'WARNING'
        ELSE 'FAIL'
    END AS status
FROM field_comparison;
Field-Level Accuracy Checks
Field	Source	Target	Tolerance	Importance
Document ID	ACDOCA.BELNR	journalEntry.documentId	Exact Match	Critical
Amount	ACDOCA.HSL	journalEntry.amountLC	±0.01	Critical
Currency	ACDOCA.RHCUR	journalEntry.localCurrency	Exact Match	Critical
Cost Centre	ACDOCA.KOSTL	journalEntry.costCentre	Exact Match	High
Posting Date	ACDOCA.BUDAT	journalEntry.postingDate	Exact Match	High
GL Account	ACDOCA.RACCT	journalEntry.glAccount	Exact Match	Critical
Profit Centre	ACDOCA.PRCTR	journalEntry.profitCentre	Exact Match	High
Fiscal Year	ACDOCA.GJAHR	journalEntry.fiscalYear	Exact Match	Critical
Document Type	BKPF.BLART	journalEntry.documentType	Exact Match	Medium
Reference	ACDOCA.XBLNR	journalEntry.referenceDocument	Exact Match	Low
Accuracy Tracking
Dimension	Daily Target	Weekly Target	Monthly Target
Accuracy	≥ 99.9%	≥ 99.9%	≥ 99.95%
Fields Checked	All critical fields	All critical fields	All fields
Sample Size	100%	100%	100%
3. Timeliness
Definition
Data is available in the target system within the agreed SLA window. This measures the end-to-end latency from SAP posting to FinSight availability.

Formula
text
Timeliness = (Records within SLA / Total Records) × 100
SLA Definitions
Priority	Data Type	SLA	Measurement
Real-time	Bank Statements, Critical GL	< 30 minutes	ODP delta to FinSight load
Batch Daily	GL, AP, AR, CO, PO, SO	< 4 hours	Daily batch to FinSight availability
Batch Monthly	Fixed Assets, Budget	< 24 hours	Monthly batch to FinSight availability
Ad-hoc	Reports, Analytics	< 15 minutes	Query response time
Validation Method
sql
-- Timeliness check SQL
SELECT 
    COUNT(*) AS total_records,
    SUM(CASE 
        WHEN (t.loaded_at - s.extracted_at) <= INTERVAL '4 hours' 
        THEN 1 ELSE 0 
    END) AS within_sla,
    SUM(CASE 
        WHEN (t.loaded_at - s.extracted_at) > INTERVAL '4 hours' 
        THEN 1 ELSE 0 
    END) AS outside_sla,
    AVG(EXTRACT(EPOCH FROM (t.loaded_at - s.extracted_at))) AS avg_seconds,
    (SUM(CASE 
        WHEN (t.loaded_at - s.extracted_at) <= INTERVAL '4 hours' 
        THEN 1 ELSE 0 
    END) / COUNT(*)) * 100 AS timeliness_pct,
    CASE 
        WHEN (SUM(CASE 
            WHEN (t.loaded_at - s.extracted_at) <= INTERVAL '4 hours' 
            THEN 1 ELSE 0 
        END) / COUNT(*)) * 100 >= 98 
        THEN 'PASS'
        WHEN (SUM(CASE 
            WHEN (t.loaded_at - s.extracted_at) <= INTERVAL '4 hours' 
            THEN 1 ELSE 0 
        END) / COUNT(*)) * 100 >= 95 
        THEN 'WARNING'
        ELSE 'FAIL'
    END AS status
FROM source_records s
JOIN target_records t ON s.id = t.source_id
WHERE s.batch_id = 'BATCH-GL-20260328-0930';
Timeliness Tracking
Batch Type	SLA	Critical Threshold	Alert Threshold
ODP Delta	30 minutes	45 minutes	35 minutes
Daily Batch	4 hours	6 hours	5 hours
Monthly Batch	24 hours	36 hours	28 hours
Ad-hoc Query	15 minutes	30 minutes	20 minutes
End-to-End Latency Breakdown
sql
-- Latency breakdown
SELECT 
    AVG(s.extracted_at - s.created_at) AS source_to_extract,
    AVG(t.transformed_at - s.extracted_at) AS extract_to_transform,
    AVG(t.loaded_at - t.transformed_at) AS transform_to_load,
    AVG(t.loaded_at - s.created_at) AS end_to_end_latency
FROM source_records s
JOIN target_records t ON s.id = t.source_id
WHERE s.batch_id = 'BATCH-GL-20260328-0930';
4. Consistency
Definition
Data values are consistent across related datasets, reference data, and business rules. No logical contradictions should exist.

Formula
text
Consistency = (Consistent Records / Total Records) × 100
Consistency Checks
Check ID	Check Type	Description	Rule
CONS-001	Debit-Credit Balance	Debits equal credits per document	SUM(debits) = SUM(credits)
CONS-002	Cross-Reference	Records reference valid master data	Cost centre exists in master
CONS-003	Currency Consistency	All amounts use valid currencies	Currency code in ISO 4217
CONS-004	Period Consistency	Periods align with fiscal year	Period between 001-016
CONS-005	Data Type Consistency	Fields have correct data types	Amount is numeric, date is valid
CONS-006	Unique Key Consistency	No duplicate business keys	Document ID is unique
CONS-007	Referential Integrity	Foreign keys reference valid records	GL account exists in chart
CONS-008	GST Consistency	GST breakdown matches tax code	CGST+SGST=IGST for inter-state
CONS-009	Fiscal Year Consistency	Period matches fiscal year	Period 001 in April of fiscal year
Validation Method
sql
-- Debit-Credit Balance Check (CONS-001)
SELECT 
    s.document_id,
    SUM(CASE WHEN s.dr_cr = 'S' THEN s.amount ELSE 0 END) AS total_debit,
    SUM(CASE WHEN s.dr_cr = 'H' THEN s.amount ELSE 0 END) AS total_credit,
    ABS(SUM(CASE WHEN s.dr_cr = 'S' THEN s.amount ELSE 0 END) - 
        SUM(CASE WHEN s.dr_cr = 'H' THEN s.amount ELSE 0 END)) AS balance,
    CASE 
        WHEN ABS(SUM(CASE WHEN s.dr_cr = 'S' THEN s.amount ELSE 0 END) - 
                  SUM(CASE WHEN s.dr_cr = 'H' THEN s.amount ELSE 0 END)) <= 0.01 
        THEN 'PASS'
        ELSE 'FAIL'
    END AS status
FROM source_records s
WHERE s.batch_id = 'BATCH-GL-20260328-0930'
GROUP BY s.document_id;

-- Cross-Reference Validity Check (CONS-002)
SELECT 
    COUNT(*) AS total_references,
    SUM(CASE WHEN cc.cost_centre_id IS NOT NULL THEN 1 ELSE 0 END) AS valid_references,
    SUM(CASE WHEN cc.cost_centre_id IS NULL THEN 1 ELSE 0 END) AS invalid_references,
    (SUM(CASE WHEN cc.cost_centre_id IS NOT NULL THEN 1 ELSE 0 END) / COUNT(*)) * 100 AS validity_rate
FROM source_records s
LEFT JOIN cost_centre_master cc ON s.cost_centre = cc.cost_centre_id
WHERE s.batch_id = 'BATCH-GL-20260328-0930';

-- Currency Consistency Check (CONS-003)
SELECT 
    DISTINCT s.currency,
    CASE 
        WHEN s.currency IN ('INR', 'USD', 'EUR', 'GBP', 'JPY', 'AUD', 'CAD', 'CHF', 'CNY', 'HKD') 
        THEN 'VALID'
        ELSE 'INVALID'
    END AS currency_status
FROM source_records s
WHERE s.batch_id = 'BATCH-GL-20260328-0930';
Consistency Tracking
Dimension	Daily Target	Weekly Target	Monthly Target
Consistency	100%	100%	100%
Checks Performed	9 checks	9 checks	9 checks
Sample Size	100%	100%	100%
Reconciliation Scorecard
Overall Score Calculation
text
Overall Score = (Completeness × 0.30) + (Accuracy × 0.30) + (Timeliness × 0.20) + (Consistency × 0.20)
Weighting Rationale
Dimension	Weight	Rationale
Completeness	30%	Data loss is unacceptable
Accuracy	30%	Incorrect data is worse than missing data
Timeliness	20%	Data must be available when needed
Consistency	20%	Data must be logically consistent
Scoring Matrix
Score Range	Status	Color	Action	Communication
99.5% - 100%	EXCELLENT	🟢 Green	Proceed normally	Status update only
98% - 99.49%	GOOD	🟡 Yellow	Monitor closely	Notify Integration Lead
95% - 97.99%	WARNING	🟠 Orange	Investigate immediately	Notify VP IT
< 95%	FAIL	🔴 Red	Block/Pause	Notify CFO + VP IT
Scorecard Template
json
{
    "batch_id": "BATCH-GL-20260328-0930",
    "domain": "General Ledger",
    "reconciliation_timestamp": "2026-03-28T09:35:00+05:30",
    "dimensions": {
        "completeness": {
            "score": 99.98,
            "status": "EXCELLENT",
            "details": "6,241/6,243 records matched",
            "weight": 0.30,
            "weighted_score": 29.99
        },
        "accuracy": {
            "score": 100.00,
            "status": "EXCELLENT",
            "details": "All 50 fields match",
            "weight": 0.30,
            "weighted_score": 30.00
        },
        "timeliness": {
            "score": 99.95,
            "status": "EXCELLENT",
            "details": "45 seconds within SLA",
            "weight": 0.20,
            "weighted_score": 19.99
        },
        "consistency": {
            "score": 100.00,
            "status": "EXCELLENT",
            "details": "All 9 consistency checks passed",
            "weight": 0.20,
            "weighted_score": 20.00
        }
    },
    "overall_score": 99.98,
    "overall_status": "EXCELLENT",
    "recommendation": "Proceed with load",
    "summary": {
        "total_records": 6243,
        "records_processed": 6241,
        "records_failed": 4,
        "records_skipped": 2,
        "dlq_depth": 17,
        "processing_duration": "2m45s",
        "throughput": "37.8 records/sec"
    }
}
Reconciliation Dashboard KPIs
Daily KPIs
KPI	Target	Alert Threshold	Status
Completeness	≥ 99.5%	< 99.5%	🟢
Accuracy	≥ 99.9%	< 99.9%	🟢
Timeliness	≥ 98%	< 98%	🟢
Consistency	100%	< 100%	🟢
DLQ Depth	< 50	> 50	🟢
Processing Time	< 1 hour	> 2 hours	🟢
Weekly KPIs
KPI	Target	Alert Threshold
Zero-Break Days	≥ 6	< 5
Average Score	≥ 99.5%	< 99%
DLQ Resolution Rate	≥ 80%	< 70%
Monthly KPIs
KPI	Target	Alert Threshold
Overall Score	≥ 99.8%	< 99.5%
Data Integrity	100%	< 100%
Audit Compliance	100%	< 100%
Reconciliation Failure Management
Break Severity Levels
Level	Description	Example	Action
Level 1	Minor break, auto-correctable	Date format correction	Auto-fix, log warning
Level 2	Moderate break, manual review	Cost centre not found	Route to DLQ, P3 alert
Level 3	Major break, immediate action	Debit-credit mismatch	Block, P2 alert
Level 4	Critical break, system stop	Data corruption	Stop pipeline, P1 alert
Break Resolution Workflow
text
┌─────────────────┐
│  Break Detected │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Severity       │
│  Assessment     │
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
 Level 1    Level 2-4
    │         │
    ▼         ▼
 Auto-fix   Route to
            DLQ
              │
              ▼
        ┌─────────────┐
        │  Manual     │
        │  Review     │
        └──────┬──────┘
               │
         ┌─────┴─────┐
         │           │
         ▼           ▼
    Corrected    Permanent
         │        Failure
         ▼           │
    Reprocess     Resolve
                  in DLQ
Changelog
