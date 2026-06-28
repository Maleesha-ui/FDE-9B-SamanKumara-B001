# Currency Conversion Specification

## Overview

Currency conversion is required for multi-currency transactions from SAP to FinSight. All amounts must be converted to the company code local currency (INR for Indian entities).

## Source Currency Data

| Field | Table | Description |
|-------|-------|-------------|
| `RWCUR` | ACDOCA | Transaction currency |
| `RHCUR` | ACDOCA | Local currency (company code currency) |
| `WSL` | ACDOCA | Amount in transaction currency |
| `HSL` | ACDOCA | Amount in local currency |
| `UKURS` | TCURR | Exchange rate |

## Exchange Rate Lookup

```sql
SELECT UKURS, FFACT, TFACT
FROM TCURR
WHERE KURST = 'M'  -- Average rate type
  AND FCURR = :transactionCurrency
  AND TCURR = :localCurrency
  AND GDATU = :postingDate