# Domain 2: Accounts Payable Mappings

## Data Domain: Accounts Payable (AP)
**Source Tables:** BSIK, LFA1, LFC1  
**Target Objects:** Vendor, VendorOpenItem  
**Total Mappings:** 8

---

## MAP-AP-001: Vendor ID

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-AP-001 |
| **Data Domain** | Accounts Payable |
| **Source Field** | `LFA1.LIFNR` |
| **Target Field** | `vendor.vendorId` |
| **Transformation Rule** | Direct mapping |
| **Validation** | NOT NULL, Max length: 30 |
| **Error Handling** | If NULL, skip vendor |
| **Business Context** | Unique vendor identifier in SAP |
| **Example** | `V0001234` → `V0001234` |

---

## MAP-AP-002: Vendor Name

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-AP-002 |
| **Data Domain** | Accounts Payable |
| **Source Field** | `LFA1.NAME1` |
| **Target Field** | `vendor.vendorName` |
| **Transformation Rule** | Direct mapping, trim whitespace |
| **Validation** | NOT NULL, Max length: 100 |
| **Error Handling** | If NULL, use empty string and log warning |
| **Business Context** | Vendor legal name |
| **Example** | `ABC Suppliers Pvt Ltd` → `ABC Suppliers Pvt Ltd` |

---

## MAP-AP-003: GST Number

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-AP-003 |
| **Data Domain** | Accounts Payable |
| **Source Field** | `LFA1.STCD1` |
| **Target Field** | `vendor.gstNumber` |
| **Transformation Rule** | Direct mapping, validate GST format |
| **Validation** | Pattern: `^[0-9]{2}[A-Z]{5}[0-9]{4}[A-Z]{1}[1-9A-Z]{1}[Z]{1}[0-9A-Z]{1}$` |
| **Error Handling** | If invalid, route to DLQ with ERR-MAP-003 |
| **Business Context** | GST registration number for India compliance |
| **Example** | `27AABCU1234D1Z1` → `27AABCU1234D1Z1` |

---

## MAP-AP-004: PAN Number

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-AP-004 |
| **Data Domain** | Accounts Payable |
| **Source Field** | `LFA1.STCD2` |
| **Target Field** | `vendor.panNumber` |
| **Transformation Rule** | Direct mapping |
| **Validation** | Pattern: `^[A-Z]{5}[0-9]{4}[A-Z]{1}$` |
| **Error Handling** | If invalid, log warning but process (optional field) |
| **Business Context** | PAN number for TDS compliance |
| **Example** | `AABCU1234D` → `AABCU1234D` |

---

## MAP-AP-005: Reconciliation Account

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-AP-005 |
| **Data Domain** | Accounts Payable |
| **Source Field** | `LFC1.AKONT` |
| **Target Field** | `vendor.reconciliationAccount` |
| **Transformation Rule** | Direct mapping |
| **Validation** | NOT NULL, Must exist in chart of accounts |
| **Error Handling** | If invalid, route to DLQ |
| **Business Context** | G/L account for vendor reconciliation |
| **Example** | `200000` → `200000` |

---

## MAP-AP-006: Aging Bucket

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-AP-006 |
| **Data Domain** | Accounts Payable |
| **Source Field** | `BSIK.AUGDT`, `BSIK.BUDAT`, current date |
| **Target Field** | `vendorOpenItem.agingBucket` |
| **Transformation Rule** | Calculate days overdue: `CURRENT_DATE - MAX(AUGDT, BUDAT + terms)` |
| **Validation** | Must be one of: [0-30, 31-60, 61-90, 91-120, 120+] |
| **Error Handling** | If calculation fails, use default bucket '0-30' |
| **Business Context** | Aging of vendor payables for cash management |
| **Example** | `45 days overdue` → `31-60` |

---

## MAP-AP-007: Clearing Status

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-AP-007 |
| **Data Domain** | Accounts Payable |
| **Source Field** | `BSIK.AUGDT`, `BSIK.AUGBL` |
| **Target Field** | `vendorOpenItem.clearingStatus` |
| **Transformation Rule** | `IF AUGDT IS NOT NULL AND AUGBL IS NOT NULL THEN 'CLEARED' ELSE 'OPEN'` |
| **Validation** | Must be [OPEN, CLEARED, PARTIALLY_CLEARED] |
| **Error Handling** | Use 'OPEN' as default |
| **Business Context** | Whether vendor invoice has been paid |
| **Example** | `AUGDT IS NULL` → `OPEN` |

---

## MAP-AP-008: Remaining Amount

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-AP-008 |
| **Data Domain** | Accounts Payable |
| **Source Field** | `BSIK.DMBTR`, `BSIK.WRBTR` |
| **Target Field** | `vendorOpenItem.remainingAmount` |
| **Transformation Rule** | `IF cleared THEN 0 ELSE original amount - payments` |
| **Validation** | Must be >= 0 |
| **Error Handling** | If negative, set to 0 and log warning |
| **Business Context** | Outstanding amount still to be paid |
| **Example** | `150000.00 - 50000.00` → `100000.00` |