# Error Classification Taxonomy

## Overview

This document defines the error classification framework for the SAP to FinSight integration pipeline. All errors are categorized to determine appropriate handling strategies.

---

## Error Categories

### 1. TRANSIENT ERRORS

Temporary failures that may resolve with retry.

| Error Code | Description | Retry Strategy |
|------------|-------------|----------------|
| ERR-EXT-001 | SAP RFC connection timeout after 30 seconds | Exponential backoff, max 3 attempts |
| ERR-EXT-002 | SAP RFC connection pool exhausted (50 connections) | Wait 60s, retry, trigger pool alert |
| ERR-EXT-004 | ODP delta request returned empty despite known changes | Retry once, check ODP monitor (RSA1) |
| ERR-LOAD-001 | FinSight API HTTP 429 Too Many Requests | Honour Retry-After header |
| ERR-LOAD-002 | FinSight API HTTP 503 Service Unavailable | Linear backoff 60s, max 5 attempts |
| ERR-SYS-001 | Kafka broker unreachable | Trigger circuit breaker, alert P1 |

### 2. PERMANENT ERRORS

Failures that will not resolve with retry.

| Error Code | Description | Action |
|------------|-------------|--------|
| ERR-EXT-003 | ODP subscription invalidated (provider structure changed) | Alert integration team, halt extraction |
| ERR-MAP-006 | Transformation rule execution error (division by zero, overflow) | Log full context, route to DLQ, alert engineering |
| ERR-LOAD-003 | FinSight API HTTP 422 Business Rule Violation | Route to business exception queue, no retry |
| ERR-LOAD-004 | FinSight API HTTP 409 Duplicate Entry | Compare payloads, skip if identical, alert if different |
| ERR-LOAD-005 | FinSight API HTTP 401 Authentication Failure | Refresh token once, if still fails alert security |

### 3. DATA QUALITY ERRORS

Data quality issues requiring manual intervention.

| Error Code | Description | Action |
|------------|-------------|--------|
| ERR-MAP-001 | Source field is NULL but target requires NOT NULL | Apply default if defined, else route to DLQ |
| ERR-MAP-002 | Date format invalid (not YYYYMMDD or valid date) | Attempt format correction, if fails route to DLQ |
| ERR-MAP-003 | Referenced cost centre not found in master data | Route to business exception queue, notify functional team |
| ERR-MAP-004 | Currency code not found in ISO 4217 lookup | Check for common typos, else DLQ |
| ERR-MAP-005 | Exchange rate not found for currency pair and date | Use nearest available date rate with stale-rate flag |
| ERR-RECON-001 | Record count mismatch between source and target | Generate break report, trigger investigation |
| ERR-RECON-002 | Checksum variance exceeds tolerance threshold | Generate break report, escalate to data steward |
| ERR-RECON-003 | Referential integrity violation in target | Identify missing master data, trigger master data resync |

### 4. SYSTEM ERRORS

Infrastructure and system-level failures.

| Error Code | Description | Action |
|------------|-------------|--------|
| ERR-SYS-001 | Kafka broker unreachable | Trigger circuit breaker, alert infrastructure team P1 |
| ERR-SYS-002 | Disk space below 10% on processing node | Alert infrastructure, archive old logs, request expansion |
| ERR-SYS-003 | Memory exhaustion in transformation engine | Restart container, scale up resources |

---

## Error Priority Matrix

| Priority | Description | Response Time | Notification |
|----------|-------------|---------------|--------------|
| **P1** | Critical - System down, data loss risk | < 15 minutes | PagerDuty + SMS + Email |
| **P2** | High - Major functionality affected | < 1 hour | PagerDuty + Email |
| **P3** | Medium - Minor functionality affected | < 4 hours | Email |
| **P4** | Low - Cosmetic issues, warnings | < 24 hours | Log only |

---

## Error Classification Flowchart
