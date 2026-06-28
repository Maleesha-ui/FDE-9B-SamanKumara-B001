# Domain 3: Accounts Receivable Mappings

## Data Domain: Accounts Receivable (AR)
**Source Tables:** BSID, KNA1, KNC1  
**Target Objects:** Customer, CustomerOpenItem  
**Total Mappings:** 8

---

## MAP-AR-001: Customer ID

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-AR-001 |
| **Data Domain** | Accounts Receivable |
| **Source Field** | `KNA1.KUNNR` |
| **Target Field** | `customer.customerId` |
| **Transformation Rule** | Direct mapping |
| **Validation** | NOT NULL, Max length: 30 |
| **Error Handling** | If NULL, skip customer |
| **Business Context** | Unique customer identifier in SAP |
| **Example** | `C0001234` → `C0001234` |

---

## MAP-AR-002: Customer Name

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-AR-002 |
| **Data Domain** | Accounts Receivable |
| **Source Field** | `KNA1.NAME1` |
| **Target Field** | `customer.customerName` |
| **Transformation Rule** | Direct mapping, trim whitespace |
| **Validation** | NOT NULL, Max length: 100 |
| **Error Handling** | If NULL, use empty string |
| **Business Context** | Customer legal name |
| **Example** | `XYZ Trading Ltd` → `XYZ Trading Ltd` |

---

## MAP-AR-003: GST Number

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-AR-003 |
| **Data Domain** | Accounts Receivable |
| **Source Field** | `KNA1.STCD1` |
| **Target Field** | `customer.gstNumber` |
| **Transformation Rule** | Direct mapping, validate GST format |
| **Validation** | Pattern: `^[0-9]{2}[A-Z]{5}[0-9]{4}[A-Z]{1}[1-9A-Z]{1}[Z]{1}[0-9A-Z]{1}$` |
| **Error Handling** | If invalid, route to DLQ with ERR-MAP-003 |
| **Business Context** | GST registration number for India compliance |
| **Example** | `07AABCU1234D1Z1` → `07AABCU1234D1Z1` |

---

## MAP-AR-004: Credit Limit

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-AR-004 |
| **Data Domain** | Accounts Receivable |
| **Source Field** | `KNC1.KLIME` |
| **Target Field** | `customer.creditLimit` |
| **Transformation Rule** | Direct mapping |
| **Validation** | Must be >= 0 |
| **Error Handling** | If negative, set to 0 and log warning |
| **Business Context** | Credit limit assigned to customer |
| **Example** | `500000.00` → `500000.00` |

---

## MAP-AR-005: Available Credit

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-AR-005 |
| **Data Domain** | Accounts Receivable |
| **Source Field** | `KNC1.KLIME`, `KNC1.KLIME` (used) |
| **Target Field** | `customer.availableCredit` |
| **Transformation Rule** | `creditLimit - creditLimitUsed` |
| **Validation** | Must be >= 0 |
| **Error Handling** | If negative, set to 0 and log warning |
| **Business Context** | Remaining credit available for customer |
| **Example** | `500000.00 - 250000.00` → `250000.00` |

---

## MAP-AR-006: Dunning Level

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-AR-006 |
| **Data Domain** | Accounts Receivable |
| **Source Field** | `BSID.MABER` (dunning indicator) |
| **Target Field** | `customerOpenItem.dunningLevel` |
| **Transformation Rule** | Map SAP dunning indicator to numeric level (0-3) |
| **Validation** | Must be [0, 1, 2, 3] |
| **Error Handling** | If invalid, use 0 |
| **Business Context** | Collections status; higher levels indicate more urgent collection |
| **Example** | `M1` → `1` |

---

## MAP-AR-007: Aging Bucket

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-AR-007 |
| **Data Domain** | Accounts Receivable |
| **Source Field** | `BSID.AUGDT`, `BSID.BUDAT` |
| **Target Field** | `customerOpenItem.agingBucket` |
| **Transformation Rule** | Calculate days overdue: `CURRENT_DATE - MAX(AUGDT, BUDAT + terms)` |
| **Validation** | Must be one of: [0-30, 31-60, 61-90, 91-120, 120+] |
| **Error Handling** | If calculation fails, use default bucket '0-30' |
| **Business Context** | Aging of customer receivables for collections |
| **Example** | `45 days overdue` → `31-60` |

---

## MAP-AR-008: Clearing Status

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-AR-008 |
| **Data Domain** | Accounts Receivable |
| **Source Field** | `BSID.AUGDT`, `BSID.AUGBL` |
| **Target Field** | `customerOpenItem.clearingStatus` |
| **Transformation Rule** | `IF AUGDT IS NOT NULL AND AUGBL IS NOT NULL THEN 'CLEARED' ELSE 'OPEN'` |
| **Validation** | Must be [OPEN, CLEARED, PARTIALLY_CLEARED] |
| **Error Handling** | Use 'OPEN' as default |
| **Business Context** | Whether customer invoice has been paid |
| **Example** | `AUGDT IS NOT NULL` → `CLEARED` |