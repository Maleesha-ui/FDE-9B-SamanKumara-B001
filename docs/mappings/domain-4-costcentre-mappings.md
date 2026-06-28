# Domain 4: Cost Centre Mappings

## Data Domain: Cost Centre (COST)
**Source Tables:** CSKS, CSKT  
**Target Objects:** CostCentre  
**Total Mappings:** 6

---

## MAP-COST-001: Cost Centre ID

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-COST-001 |
| **Data Domain** | Cost Centre |
| **Source Field** | `CSKS.KOSTL` |
| **Target Field** | `costCentre.costCentreId` |
| **Transformation Rule** | `LTRIM(KOSTL, '0')` |
| **Validation** | NOT NULL, Max length: 20 |
| **Error Handling** | If NULL, skip cost centre |
| **Business Context** | Unique cost centre identifier |
| **Example** | `000001000` → `1000` |

---

## MAP-COST-002: Cost Centre Name

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-COST-002 |
| **Data Domain** | Cost Centre |
| **Source Field** | `CSKT.KTEXT` |
| **Target Field** | `costCentre.name` |
| **Transformation Rule** | Direct mapping by language (SPRAS = 'EN') |
| **Validation** | NOT NULL, Max length: 50 |
| **Error Handling** | If NULL, use costCentreId as name |
| **Business Context** | Descriptive name for cost centre |
| **Example** | `Production Department` → `Production Department` |

---

## MAP-COST-003: Long Description

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-COST-003 |
| **Data Domain** | Cost Centre |
| **Source Field** | `CSKT.LTEXT` |
| **Target Field** | `costCentre.longText` |
| **Transformation Rule** | Direct mapping by language (SPRAS = 'EN') |
| **Validation** | Max length: 200 |
| **Error Handling** | If NULL, use name as fallback |
| **Business Context** | Detailed description of cost centre purpose |
| **Example** | `Main Production Department - Plant PL01` |

---

## MAP-COST-004: Valid From Date

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-COST-004 |
| **Data Domain** | Cost Centre |
| **Source Field** | `CSKS.DATAB` |
| **Target Field** | `costCentre.validFrom` |
| **Transformation Rule** | `FORMAT(DATAB, 'YYYY-MM-DD')` |
| **Validation** | NOT NULL, Must be valid date |
| **Error Handling** | If invalid, use current date |
| **Business Context** | Start date of cost centre validity |
| **Example** | `20260101` → `2026-01-01` |

---

## MAP-COST-005: Valid To Date

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-COST-005 |
| **Data Domain** | Cost Centre |
| **Source Field** | `CSKS.DATBI` |
| **Target Field** | `costCentre.validTo` |
| **Transformation Rule** | `FORMAT(DATBI, 'YYYY-MM-DD')` |
| **Validation** | NOT NULL, Must be valid date |
| **Error Handling** | If invalid, use '9999-12-31' |
| **Business Context** | End date of cost centre validity |
| **Example** | `99991231` → `9999-12-31` |

---

## MAP-COST-006: Hierarchy Flattening (7 Levels)

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-COST-006 |
| **Data Domain** | Cost Centre |
| **Source Field** | `SETNODE`, `SETLEAF` |
| **Target Field** | `costCentre.hierarchy.level1..level7` |
| **Transformation Rule** | Flatten 7-level hierarchy from set structure |
| **Validation** | All levels must be populated or NULL |
| **Error Handling** | If hierarchy incomplete, fill with NULL |
| **Business Context** | Cost centre reporting hierarchy for roll-ups |
| **Example** | `Company → Division → Plant → Dept → Section → Team → Cost Centre` |

---

## MAP-COST-007: Responsible Person

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-COST-007 |
| **Data Domain** | Cost Centre |
| **Source Field** | `CSKS.VERAK` |
| **Target Field** | `costCentre.responsiblePerson` |
| **Transformation Rule** | Direct mapping |
| **Validation** | Max length: 30 |
| **Error Handling** | If NULL, leave empty |
| **Business Context** | Person responsible for cost centre budget |
| **Example** | `MANAGER01` → `MANAGER01` |