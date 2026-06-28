# Domain 1: General Ledger Mappings

## Data Domain: General Ledger (GL)
**Source Tables:** ACDOCA, BKPF, BSEG  
**Target Objects:** JournalEntry  
**Total Mappings:** 12

---

## MAP-GL-001: Document ID

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-GL-001 |
| **Data Domain** | General Ledger |
| **Source Field** | `ACDOCA.BUKRS`, `ACDOCA.GJAHR`, `ACDOCA.BELNR` |
| **Target Field** | `journalEntry.documentId` |
| **Transformation Rule** | `CONCAT(BUKRS, '-', GJAHR, '-', LTRIM(BELNR, '0'))` |
| **Validation** | NOT NULL, Regex: `^[A-Z0-9]{2,6}-\d{4}-\d{1,10}$` |
| **Error Handling** | If invalid format, log error and route to DLQ |
| **Business Context** | Unique document identifier across company codes and fiscal years |
| **Example** | `MC01-2026-0000012345` → `MC01-2026-12345` |

---

## MAP-GL-002: Posting Date

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-GL-002 |
| **Data Domain** | General Ledger |
| **Source Field** | `ACDOCA.BUDAT` |
| **Target Field** | `journalEntry.postingDate` |
| **Transformation Rule** | `FORMAT(BUDAT, 'YYYY-MM-DD')` from SAP internal `YYYYMMDD` |
| **Validation** | NOT NULL, Must be within fiscal year +/- 1 month |
| **Error Handling** | If invalid date, attempt format correction; if fails, route to DLQ |
| **Business Context** | Date when transaction is posted in SAP; used for period mapping |
| **Example** | `20260328` → `2026-03-28` |

---

## MAP-GL-003: Document Date

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-GL-003 |
| **Data Domain** | General Ledger |
| **Source Field** | `ACDOCA.BLDAT` |
| **Target Field** | `journalEntry.documentDate` |
| **Transformation Rule** | `FORMAT(BLDAT, 'YYYY-MM-DD')` |
| **Validation** | NOT NULL, Must be <= postingDate + 30 days |
| **Error Handling** | If document date > posting date + 30 days, log warning but process |
| **Business Context** | Original document date from source (invoice, payment, etc.) |
| **Example** | `20260328` → `2026-03-28` |

---

## MAP-GL-004: Document Type

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-GL-004 |
| **Data Domain** | General Ledger |
| **Source Field** | `BKPF.BLART` |
| **Target Field** | `journalEntry.documentType` |
| **Transformation Rule** | Direct mapping with SAP document type lookup |
| **Validation** | Must be valid SAP document type (SA, RV, KR, KA, WA, AB, DZ, KZ) |
| **Error Handling** | If invalid, route to DLQ with error code ERR-MAP-006 |
| **Business Context** | Classification of financial document; drives processing rules |
| **Example** | `SA` → `SA` |

---

## MAP-GL-005: Company Code

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-GL-005 |
| **Data Domain** | General Ledger |
| **Source Field** | `ACDOCA.BUKRS` |
| **Target Field** | `journalEntry.companyCode` |
| **Transformation Rule** | Direct mapping |
| **Validation** | NOT NULL, Must be in [MC01, MC02, MC03] |
| **Error Handling** | If invalid company code, route to DLQ |
| **Business Context** | Legal entity; determines tenant routing in FinSight |
| **Example** | `MC01` → `MC01` |

---

## MAP-GL-006: Fiscal Year

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-GL-006 |
| **Data Domain** | General Ledger |
| **Source Field** | `ACDOCA.GJAHR` |
| **Target Field** | `journalEntry.fiscalYear` |
| **Transformation Rule** | Direct mapping (4-digit year) |
| **Validation** | NOT NULL, Pattern: `^20[0-9]{2}$` |
| **Error Handling** | If invalid, route to DLQ |
| **Business Context** | Indian fiscal year (April-March) |
| **Example** | `2026` → `2026` |

---

## MAP-GL-007: Posting Period

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-GL-007 |
| **Data Domain** | General Ledger |
| **Source Field** | `ACDOCA.MONAT` |
| **Target Field** | `journalEntry.postingPeriod` |
| **Transformation Rule** | Map SAP period 001-012 to calendar month; 013-016 map to 012 with specialPeriod=true |
| **Validation** | Range 001-016, Flag special periods |
| **Error Handling** | If period outside range, route to DLQ |
| **Business Context** | Indian fiscal period; special periods (013-016) are for year-end adjustments |
| **Example** | `003` → `003` (March period) |

---

## MAP-GL-008: GL Account

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-GL-008 |
| **Data Domain** | General Ledger |
| **Source Field** | `ACDOCA.RACCT` |
| **Target Field** | `journalEntry.glAccount` |
| **Transformation Rule** | `LTRIM(RACCT, '0')` then lookup in analytics chart of accounts |
| **Validation** | Must exist in target chart of accounts master |
| **Error Handling** | If account not found, log and route to DLQ |
| **Business Context** | Maps SAP G/L account to FinSight account hierarchy |
| **Example** | `00400000` → `400000` |

---

## MAP-GL-009: Amount in Local Currency

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-GL-009 |
| **Data Domain** | General Ledger |
| **Source Field** | `ACDOCA.HSL` |
| **Target Field** | `journalEntry.amountLC` |
| **Transformation Rule** | `DECIMAL(HSL, 2)` in group currency |
| **Validation** | NOT NULL, Range: `-999999999.99` to `999999999.99` |
| **Error Handling** | If NULL, route to DLQ; If range violation, log and route to DLQ |
| **Business Context** | Amount in company code local currency (INR for Indian entities) |
| **Example** | `1500000.00` → `1500000.00` |

---

## MAP-GL-010: Amount in Transaction Currency

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-GL-010 |
| **Data Domain** | General Ledger |
| **Source Field** | `ACDOCA.WSL` |
| **Target Field** | `journalEntry.amountTC` |
| **Transformation Rule** | `DECIMAL(WSL, 2)` in transaction currency |
| **Validation** | NOT NULL, Same range as amountLC |
| **Error Handling** | If NULL, use amountLC as fallback |
| **Business Context** | Amount in original transaction currency (USD, EUR, etc.) |
| **Example** | `1500000.00` → `1500000.00` |

---

## MAP-GL-011: Local Currency

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-GL-011 |
| **Data Domain** | General Ledger |
| **Source Field** | `ACDOCA.RHCUR` |
| **Target Field** | `journalEntry.localCurrency` |
| **Transformation Rule** | Direct mapping, validate ISO 4217 |
| **Validation** | Must be valid 3-letter ISO 4217 code |
| **Error Handling** | If invalid, check common typos (e.g., RS vs INR), else DLQ |
| **Business Context** | Company code currency; for Indian entities, typically INR |
| **Example** | `INR` → `INR` |

---

## MAP-GL-012: Cost Centre

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-GL-012 |
| **Data Domain** | General Ledger |
| **Source Field** | `ACDOCA.KOSTL` |
| **Target Field** | `journalEntry.costCentre` |
| **Transformation Rule** | `LTRIM(KOSTL, '0')` then validate against CC master |
| **Validation** | If present, must exist in target cost centre dimension |
| **Error Handling** | If not found, route to business exception queue; notify functional team |
| **Business Context** | Cost centre where cost is incurred |
| **Example** | `000001000` → `1000` |