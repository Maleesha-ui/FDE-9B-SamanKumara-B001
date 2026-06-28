# Domain 5: Profit Centre Mappings

## Data Domain: Profit Centre (PC)
**Source Tables:** CEPC, CEPCT  
**Target Objects:** ProfitCentre  
**Total Mappings:** 4

---

## MAP-PC-001: Profit Centre ID

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-PC-001 |
| **Data Domain** | Profit Centre |
| **Source Field** | `CEPC.PRCTR` |
| **Target Field** | `profitCentre.profitCentreId` |
| **Transformation Rule** | `LTRIM(PRCTR, '0')` |
| **Validation** | NOT NULL, Max length: 20 |
| **Error Handling** | If NULL, skip profit centre |
| **Business Context** | Unique profit centre identifier |
| **Example** | `000001000` → `1000` |

---

## MAP-PC-002: Profit Centre Name

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-PC-002 |
| **Data Domain** | Profit Centre |
| **Source Field** | `CEPCT.KTEXT` |
| **Target Field** | `profitCentre.name` |
| **Transformation Rule** | Direct mapping by language (SPRAS = 'EN') |
| **Validation** | NOT NULL, Max length: 50 |
| **Error Handling** | If NULL, use profitCentreId as name |
| **Business Context** | Descriptive name for profit centre |
| **Example** | `Automotive Division` → `Automotive Division` |

---

## MAP-PC-003: Segment

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-PC-003 |
| **Data Domain** | Profit Centre |
| **Source Field** | `CEPC.SEGMENT` |
| **Target Field** | `profitCentre.segment` |
| **Transformation Rule** | Direct mapping |
| **Validation** | Must be valid segment code |
| **Error Handling** | If NULL, use default segment '01' |
| **Business Context** | Business segment for external reporting |
| **Example** | `01` → `01` |

---

## MAP-PC-004: Hierarchy Flattening (3 Levels)

| Field | Value |
|-------|-------|
| **Mapping ID** | MAP-PC-004 |
| **Data Domain** | Profit Centre |
| **Source Field** | `SETNODE`, `SETLEAF` |
| **Target Field** | `profitCentre.hierarchy.level1..level3` |
| **Transformation Rule** | Flatten 3-level hierarchy from set structure |
| **Validation** | All levels must be populated or NULL |
| **Error Handling** | If hierarchy incomplete, fill with NULL |
| **Business Context** | Profit centre reporting hierarchy for segment reporting |
| **Example** | `Division → Business Unit → Profit Centre` |