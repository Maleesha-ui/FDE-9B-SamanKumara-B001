# Functional Test Scenarios

## Overview

This document defines 10 functional test scenarios for the SAP to FinSight integration pipeline. Each scenario validates specific functionality of the integration.

---

## Test Environment Setup

| Environment | Purpose | Configuration |
|-------------|---------|---------------|
| **DEV** | Development testing | SAP sandbox + FinSight dev tenant |
| **QA** | Integration testing | SAP QA + FinSight QA tenant |
| **UAT** | User acceptance testing | SAP UAT + FinSight UAT tenant |

---

## TST-FNC-001: Happy Path GL Extraction

| Attribute | Value |
|-----------|-------|
| **Test ID** | TST-FNC-001 |
| **Title** | Happy Path GL Extraction |
| **Description** | Validate end-to-end GL journal entry extraction and loading |
| **Preconditions** | 100 GL entries posted in SAP sandbox; ODP subscription active |

### Test Steps

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Trigger ODP delta extraction | Extraction starts successfully |
| 2 | Verify Kafka message count | 100 messages published to GL topic |
| 3 | Verify transformation engine processing | All 100 records transformed |
| 4 | Verify FinSight API load | All 100 records loaded successfully |
| 5 | Run reconciliation | 100 records in target, checksums match |

### Test Data

```json
{
    "source_records": 100,
    "domains": ["GL"],
    "expected_target_records": 100,
    "expected_checksum_match": true
}
# Functional Test Scenarios

## Overview

This document defines 10 functional test scenarios for the SAP to FinSight integration pipeline. Each scenario validates specific functionality of the integration.

---

## Test Environment Setup

| Environment | Purpose | Configuration |
|-------------|---------|---------------|
| **DEV** | Development testing | SAP sandbox + FinSight dev tenant |
| **QA** | Integration testing | SAP QA + FinSight QA tenant |
| **UAT** | User acceptance testing | SAP UAT + FinSight UAT tenant |

### Test Data Prerequisites

| Data Type | Source | Purpose |
|-----------|--------|---------|
| Master Data | SAP | Cost centres, GL accounts, Vendors, Customers |
| Transaction Data | SAP | GL entries, AP/AR items, POs, SOs |
| Reference Data | SAP | Exchange rates, Fiscal periods, Tax codes |

---

## TST-FNC-001: Happy Path GL Extraction

| Attribute | Value |
|-----------|-------|
| **Test ID** | TST-FNC-001 |
| **Title** | Happy Path GL Extraction |
| **Description** | Validate end-to-end GL journal entry extraction and loading |
| **Preconditions** | 100 GL entries posted in SAP sandbox; ODP subscription active |
| **Test Type** | Positive |

### Test Steps

| Step | Action | Expected Result | Verification |
|------|--------|-----------------|--------------|
| 1 | Trigger ODP delta extraction | Extraction starts successfully | Check scheduler logs |
| 2 | Verify Kafka message count | 100 messages published to GL topic | Check Kafka consumer lag |
| 3 | Verify transformation engine processing | All 100 records transformed | Check transformation logs |
| 4 | Verify FinSight API load | All 100 records loaded successfully | Check FinSight API response |
| 5 | Run reconciliation | 100 records in target, checksums match | Check reconciliation report |

### Test Data

```json
{
    "test_case": "TST-FNC-001",
    "source_records": 100,
    "domains": ["GL"],
    "company_codes": ["MC01"],
    "expected_target_records": 100,
    "expected_checksum_match": true,
    "sla_threshold": "4 hours"
}
Test SQL Validation
sql
-- Validate source count
SELECT COUNT(*) AS source_count FROM source_records WHERE batch_id = 'BATCH-GL-20260328-0930';

-- Validate target count
SELECT COUNT(*) AS target_count FROM target_records WHERE batch_id = 'BATCH-GL-20260328-0930';

-- Validate reconciliation
SELECT 
    SUM(CASE WHEN s.amount = t.amount THEN 1 ELSE 0 END) / COUNT(*) * 100 AS accuracy_pct
FROM source_records s
JOIN target_records t ON s.id = t.source_id
WHERE s.batch_id = 'BATCH-GL-20260328-0930';
Expected Result
text
✅ All 100 records loaded successfully
✅ Reconciliation PASS - 100/100 records matched
✅ End-to-end latency < 4 hours
✅ No errors in logs
TST-FNC-002: AP Open Items Multi-Currency
Attribute	Value
Test ID	TST-FNC-002
Title	AP Open Items Multi-Currency
Description	Validate multi-currency AP open items extraction and currency conversion
Preconditions	50 AP items in INR, USD, EUR; TCURR rates loaded
Test Type	Positive
Test Steps
Step	Action	Expected Result	Verification
1	Extract AP open items for vendor V0001234	50 items extracted	Check AP extraction logs
2	Verify currency conversion	All amounts converted to INR at correct rates	Check currency conversion logs
3	Check aging buckets	Aging buckets calculated correctly	Check aging calculation
4	Validate vendor enrichment	Vendor master data correctly enriched	Check vendor enrichment
Test Data
json
{
    "test_case": "TST-FNC-002",
    "vendor_id": "V0001234",
    "records": 50,
    "currencies": ["INR", "USD", "EUR"],
    "expected_conversion_rate": {
        "USD": 83.00,
        "EUR": 90.50
    },
    "aging_date": "2026-03-28"
}
Expected Result
text
✅ All 50 AP items loaded successfully
✅ Currency conversion accurate (USD: ₹83.00, EUR: ₹90.50)
✅ Aging buckets correctly calculated
✅ Vendor data enriched successfully
TST-FNC-003: Master Data Delta Sync
Attribute	Value
Test ID	TST-FNC-003
Title	Master Data Delta Sync
Description	Validate delta synchronization of master data changes
Preconditions	Existing cost centre master in both systems
Test Type	Positive
Test Steps
Step	Action	Expected Result	Verification
1	Create new cost centre in SAP	Cost centre created successfully	Check SAP table CSKS
2	Wait one extraction cycle	Delta extraction triggered	Check scheduler logs
3	Query target system	New cost centre appears in target	Check FinSight query
4	Verify hierarchy update	Hierarchy correctly flattened	Check hierarchy columns
Test Data
json
{
    "test_case": "TST-FNC-003",
    "cost_centre": {
        "id": "9999",
        "name": "Test Cost Centre",
        "hierarchy": ["Company", "Plant", "Department", "Section", "Team", "Sub-Team", "Cost Centre"],
        "company_code": "MC01"
    },
    "extraction_cycle": "30 minutes"
}
Expected Result
text
✅ New cost centre appears in target within one cycle
✅ Hierarchy flattened correctly (7 levels)
✅ No duplicate entries created
TST-FNC-004: Multi-Company Code Batch
Attribute	Value
Test ID	TST-FNC-004
Title	Multi-Company Code Batch
Description	Validate processing of entries from multiple company codes
Preconditions	Entries from 3 company codes (MC01, MC02, MC03)
Test Type	Positive
Test Steps
Step	Action	Expected Result	Verification
1	Extract mixed batch from all company codes	All 3 company codes extracted	Check extraction logs
2	Verify tenant routing	Each company code routes to correct tenant	Check FinSight tenant mapping
3	Check no cross-contamination	MC01 data does not appear in MC02 tenant	Check FinSight queries
Test Data
json
{
    "test_case": "TST-FNC-004",
    "company_codes": ["MC01", "MC02", "MC03"],
    "records_per_code": 100,
    "tenant_mapping": {
        "MC01": "MERIDIAN-MC01",
        "MC02": "MERIDIAN-MC02",
        "MC03": "MERIDIAN-MC03"
    }
}
Expected Result
text
✅ All 3 company codes processed correctly
✅ Tenant routing accurate (no cross-contamination)
✅ Reconciliation PASS for each code
TST-FNC-005: Fiscal Period Mapping
Attribute	Value
Test ID	TST-FNC-005
Title	Fiscal Period Mapping
Description	Validate fiscal period mapping (001-016 including special periods)
Preconditions	Entries in SAP periods 001-016
Test Type	Positive
Test Steps
Step	Action	Expected Result	Verification
1	Extract entries from all periods	All 16 periods extracted	Check extraction logs
2	Verify calendar mapping	Periods 001-012 map correctly	Check period mapping table
3	Check special period flag	Periods 013-016 map to March with flag	Check special period flag
Test Data
json
{
    "test_case": "TST-FNC-005",
    "periods": ["001", "002", "003", "004", "005", "006", "007", "008", "009", "010", "011", "012", "013", "014", "015", "016"],
    "expected_month_mapping": {
        "001": "April",
        "002": "May",
        "003": "June",
        "004": "July",
        "005": "August",
        "006": "September",
        "007": "October",
        "008": "November",
        "009": "December",
        "010": "January",
        "011": "February",
        "012": "March",
        "013": "March",
        "014": "March",
        "015": "March",
        "016": "March"
    },
    "expected_special_periods": ["013", "014", "015", "016"]
}
Expected Result
text
✅ Periods 001-012 map to correct months
✅ Periods 013-016 map to March with specialPeriod=true
✅ Quarter mapping correct (Q1-Q4)
TST-FNC-006: 7-Level Hierarchy Flatten
Attribute	Value
Test ID	TST-FNC-006
Title	7-Level Hierarchy Flatten
Description	Validate flattening of 7-level cost centre hierarchy
Preconditions	Cost centre hierarchy with 7 levels in SAP
Test Type	Positive
Test Steps
Step	Action	Expected Result	Verification
1	Extract cost centre hierarchy	Hierarchy extracted from SETNODE/SETLEAF	Check hierarchy extraction
2	Verify flattened table	Flat table with Level1 through Level7	Check flattened columns
3	Check all levels	All 7 levels correctly populated	Check hierarchy integrity
Test Data
json
{
    "test_case": "TST-FNC-006",
    "hierarchy_levels": 7,
    "expected_columns": ["level1", "level2", "level3", "level4", "level5", "level6", "level7"],
    "sample_hierarchy": {
        "level1": "MC01",
        "level2": "PLANT_A",
        "level3": "PRODUCTION",
        "level4": "ASSEMBLY",
        "level5": "TEAM_1",
        "level6": "SUB_TEAM_A",
        "level7": "1000"
    }
}
Expected Result
text
✅ All 7 levels flattened correctly
✅ No NULL values for populated levels
✅ Hierarchy integrity maintained
TST-FNC-007: P2P Flow Tracing
Attribute	Value
Test ID	TST-FNC-007
Title	P2P Flow Tracing
Description	Validate procure-to-pay flow tracing across multiple documents
Preconditions	Complete P2P: PO, GR, Invoice, Payment in SAP
Test Type	Positive
Test Steps
Step	Action	Expected Result	Verification
1	Extract all 4 stages	PO, GR, Invoice, Payment extracted	Check extraction logs
2	Trace document flow	All stages linked correctly	Check document linkage
3	Verify status mapping	Statuses correctly mapped	Check status mapping
Test Data
json
{
    "test_case": "TST-FNC-007",
    "p2p_flow": {
        "po": "PO-2026-001234",
        "gr": "GR-2026-001234",
        "invoice": "INV-2026-001234",
        "payment": "PMT-2026-001234"
    },
    "expected_statuses": {
        "PO": "OPEN",
        "GR": "COMPLETED",
        "Invoice": "OPEN",
        "Payment": "CLEARED"
    }
}
Expected Result
text
✅ All 4 stages extracted
✅ Document linkage correct
✅ Statuses correctly mapped
TST-FNC-008: Bank Statement Load
Attribute	Value
Test ID	TST-FNC-008
Title	Bank Statement Load
Description	Validate bank statement loading via IDoc
Preconditions	Bank statement with 50 items: matched and unmatched
Test Type	Positive
Test Steps
Step	Action	Expected Result	Verification
1	Load statement via IDoc	IDoc processed successfully	Check IDoc processing logs
2	Transform and load	Statement transformed and loaded	Check transformation logs
3	Verify clearing status	Matched items show CLEARED, unmatched show OPEN	Check clearing status
Test Data
json
{
    "test_case": "TST-FNC-008",
    "total_items": 50,
    "matched_items": 35,
    "unmatched_items": 15,
    "expected_clearing_status": {
        "matched": "CLEARED",
        "unmatched": "OPEN"
    },
    "statement_id": "BS-2026-001234"
}
Expected Result
text
✅ Bank statement loaded successfully
✅ Matching items correctly cleared
✅ Unmatched items correctly marked OPEN
TST-FNC-009: Budget vs Actual Variance
Attribute	Value
Test ID	TST-FNC-009
Title	Budget vs Actual Variance
Description	Validate budget vs actual variance calculation
Preconditions	Budget and actuals for 5 cost centres in SAP CO
Test Type	Positive
Test Steps
Step	Action	Expected Result	Verification
1	Extract budget data	Budget data extracted	Check COSP extraction
2	Extract actual postings	Actual postings extracted	Check COSS extraction
3	Calculate variances	Variances match SAP CO report	Check variance calculation
Test Data
json
{
    "test_case": "TST-FNC-009",
    "cost_centres": ["1000", "2000", "3000", "4000", "5000"],
    "fiscal_year": "2026",
    "period": "003",
    "version": "001",
    "expected_variance_tolerance": 0.01
}
Expected Result
text
✅ Budget and actual extracted correctly
✅ Variances match SAP CO report S_ALR_87013611
✅ Variance percentage correctly calculated
TST-FNC-010: End-of-Day Full Reconciliation
Attribute	Value
Test ID	TST-FNC-010
Title	End-of-Day Full Reconciliation
Description	Validate full end-to-end reconciliation across all domains
Preconditions	Full day of data across all 12 domains
Test Type	Positive
Test Steps
Step	Action	Expected Result	Verification
1	Run all 12 extractions	All 12 domains extracted	Check extraction logs
2	Execute full reconciliation	Reconciliation across all domains	Check reconciliation logs
3	Generate report	Zero breaks across all domains	Check reconciliation report
Test Data
json
{
    "test_case": "TST-FNC-010",
    "domains": ["GL", "AP", "AR", "COST", "PROCUREMENT", "SALES", "ASSETS", "BANK", "MATERIALS", "BUDGET", "PROFIT_CENTRE", "EXCHANGE_RATES"],
    "expected_overall_score": 99.5,
    "expected_zero_breaks": true
}
Expected Result
text
✅ All 12 domains reconciled
✅ Zero breaks across all domains
✅ Full report generated
✅ Overall score > 99.5%
Test Execution Summary
Pass/Fail Criteria
Category	Pass Criteria	Fail Criteria
Functional	All steps pass, expected results met	Any step fails or unexpected result
Performance	Within SLA thresholds	Exceeds SLA thresholds
Data Integrity	No data loss, checksums match	Data loss or checksum mismatch
Test Execution Log Template
json
{
    "test_execution_id": "TST-EXEC-20260328-001",
    "test_case_id": "TST-FNC-001",
    "execution_date": "2026-03-28",
    "executed_by": "QA Team",
    "environment": "QA",
    "status": "PASS",
    "duration_seconds": 45,
    "actual_results": {
        "records_loaded": 100,
        "reconciliation_score": 100.00,
        "latency_seconds": 45
    },
    "defects_found": [],
    "attachments": ["reconciliation_report_TST-FNC-001.pdf"]
}
