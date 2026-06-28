# Data Mapping Specification Template

## Mapping ID Format
`MAP-{DOMAIN}-{NNN}`

## Field Categories
| Category | Description |
|----------|-------------|
| **Source Field** | SAP table and field name |
| **Target Field** | FinSight field name |
| **Transformation Rule** | How source becomes target |
| **Validation** | Data quality checks |
| **Error Handling** | What happens on failure |
| **Business Context** | Why this mapping exists |

## Mapping Template

| Field | Value |
|-------|-------|
| **Mapping ID** | |
| **Data Domain** | |
| **Source Field** | |
| **Target Field** | |
| **Transformation Rule** | |
| **Validation** | |
| **Error Handling** | |
| **Business Context** | |
| **Example** | |

---

## Transformation Rule Types

| Type | Description | Example |
|------|-------------|---------|
| **Direct** | Source → Target | `ACDOCA.BUKRS → companyCode` |
| **Format** | Change data format | `YYYYMMDD → YYYY-MM-DD` |
| **Lookup** | Map using reference table | `RACCT → analytics CoA` |
| **Concatenation** | Combine multiple fields | `BUKRS + BELNR → documentId` |
| **Conditional** | If-else logic | `IF STBLG IS NOT NULL → isReversal = true` |
| **Calculation** | Mathematical operation | `AMOUNT * EXCHANGE_RATE` |
| **Flattening** | Hierarchy to flat structure | `7-level hierarchy → columns` |
| **Enrichment** | Add from master data | `KOSTL → costCentreName` |

---

## Validation Rule Types

| Type | Description |
|------|-------------|
| **NOT NULL** | Field must have value |
| **Format** | Regex pattern match |
| **Range** | Value within min-max |
| **Referential** | Exists in master data |
| **Cross-Field** | Relationship between fields |
| **Business** | Domain-specific rules |