# Integration Architecture Document (D1)

## 1. Executive Summary

This document describes the integration architecture connecting Meridian Manufacturing's SAP S/4HANA ERP system with Zetheta FinSight financial analytics platform. The integration enables real-time (4-hour SLA) data flow from SAP to FinSight, eliminating the current 24-hour batch delay.

## 2. Architecture Overview

### 2.1 High-Level Architecture

### 2.2 Architecture Principles

| Principle | Description |
|-----------|-------------|
| **Decoupling** | Source and target systems are decoupled via Kafka |
| **Idempotency** | All API calls are idempotent to prevent duplicate processing |
| **Resilience** | Circuit breakers, retries, and DLQ handle failures |
| **Observability** | Full monitoring with Prometheus, Grafana, ELK |
| **Security** | OAuth 2.0, TLS, and data encryption |
| **Compliance** | RBI data localization (AWS Mumbai region) |

## 3. Component Architecture

### 3.1 Integration Components

| Component | Technology | Purpose |
|-----------|------------|---------|
| API Gateway | Kong / AWS API Gateway | Authentication, routing, rate limiting |
| Message Broker | Apache Kafka (MSK) | Event streaming, decoupling |
| Transformation Engine | Apache Spark / dbt | Data transformation, validation |
| Scheduler | Apache Airflow | Job orchestration, scheduling |
| Reconciliation Service | Python + SQL | Data reconciliation, audit |
| Monitoring Stack | Prometheus + Grafana + ELK | Observability, alerting |
| Dead Letter Queue | S3 + PostgreSQL | Failed record storage |

### 3.2 Data Domains (12 Domains)

| Domain | Source Table | Frequency | Records/Day |
|--------|--------------|-----------|-------------|
| General Ledger | ACDOCA | 30 min | ~70,000 |
| Accounts Payable | BSIK, LFA1 | 30 min | ~10,000 |
| Accounts Receivable | BSID, KNA1 | 30 min | ~8,000 |
| Cost Centre | CSKS, CSKT | Daily | ~500 |
| Profit Centre | CEPC, CEPCT | Daily | ~200 |
| Material Ledger | MBEW, MARD | Daily | ~15,000 |
| Purchase Orders | EKKO, EKPO | 30 min | ~5,000 |
| Sales Orders | VBAK, VBAP | 30 min | ~3,000 |
| Fixed Assets | ANLA, ANLZ | Monthly | ~100 |
| Bank Statements | FEBEP, FEBKO | Real-time | ~1,000 |
| Budget/Actual | COSP, COSS | Daily | ~20,000 |
| Inventory | MBEW, MARD | Daily | ~10,000 |

## 4. Data Flow

### 4.1 Real-time Flow (ODP Delta)

1. Scheduler triggers every 30 minutes
2. SAP CDS View query executed with delta token
3. Changed records published to Kafka
4. Transformation engine processes records
5. Data loaded to FinSight (idempotent API)

### 4.2 Batch Flow (Daily)

1. Scheduler triggers daily at 02:00 IST
2. Batch extraction from SAP (COSP, COSS)
3. Transformation and aggregation
4. Load to FinSight and archive to Snowflake
5. Daily reconciliation run

## 5. Error Handling

### 5.1 Error Classification

| Error Type | Description | Action |
|------------|-------------|--------|
| TRANSIENT | Temporary failures (network, timeout) | Retry with backoff |
| PERMANENT | Configuration errors, invalid data | Route to DLQ |
| DATA QUALITY | Validation failures | Route to DLQ |
| SYSTEM | Infrastructure failures | Alert, circuit break |

### 5.2 Retry Strategy

- Exponential backoff: `wait = min(cap, random(base, base * 2^attempt))`
- Base: 2 seconds, Cap: 60 seconds
- Max attempts: 3
- Circuit breaker: Open after 5 consecutive failures

## 6. Monitoring

### 6.1 Metrics (12 Panels)

| Panel | Metric | Threshold |
|-------|--------|-----------|
| Pipeline Health | Status | RED >4hrs stale |
| Throughput | Records/sec | Warning <50% baseline |
| Latency | P95 API duration | Warning >5s |
| Error Rate | % of requests | Critical >5% |
| DLQ Depth | Kafka consumer lag | Critical >500 |
| Reconciliation | Status | Break = P2 Alert |

### 6.2 Alerting (15 Rules)

| Alert | Severity | Action |
|-------|----------|--------|
| Pipeline Failure | P1 | PagerDuty |
| DLQ Depth >500 | P1 | PagerDuty |
| Reconciliation Break | P2 | Email + PagerDuty |
| Throughput Degraded | P3 | Email |
| Resource Utilization | P2 | PagerDuty |

## 7. Security

### 7.1 Authentication
- OAuth 2.0 + JWT (client credentials flow)
- Token refresh every 1 hour

### 7.2 Authorization
- RBAC for all API endpoints
- Scopes: `read:data`, `write:data`, `admin`

### 7.3 Encryption
- In-transit: TLS 1.2+
- At-rest: AES-256

## 8. Deployment

### 8.1 Environments

| Environment | Purpose | Schedule |
|-------------|---------|----------|
| DEV | Development | 24/7 |
| QA | Testing | 24/7 |
| PROD | Production | 24/7 (Maintenance: 2nd/4th Sat 22:00-06:00) |

### 8.2 Rollback Procedure
1. Stop integration jobs
2. Revert to previous version
3. Resume processing from last delta token
4. Verify reconciliation
5. Max rollback time: <15 minutes

## 9. Stakeholder Communication

| Stakeholder | Document | Frequency |
|-------------|----------|-----------|
| CFO (Ananya) | Executive Summary | Weekly |
| VP IT (Rajesh) | Technical Handoff | Monthly |
| SAP Admin (Priya) | Performance Report | Daily |
| Platform (Marcus) | API Design Review | Weekly |
| Audit (Dr. Sanjay) | Reconciliation Report | Daily |
| Operations (Fatima) | Dashboard Status | Daily |

## 10. Success Criteria

| Criterion | Target | Measurement |
|-----------|--------|-------------|
| Data Freshness | <4 hours | SAP posting to FinSight availability |
| Data Loss | 0% | Reconciliation checks |
| Reconciliation Accuracy | >99.99% | Source-Target match |
| SAP Performance Impact | <10% | CPU/Memory usage |
| SLA Compliance | >99.5% | Uptime and latency |
| Audit Trail | 100% | Data lineage tracking |