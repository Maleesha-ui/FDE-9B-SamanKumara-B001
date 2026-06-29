# Technical Design Review: SAP to FinSight Integration

**Date:** March 29, 2026  
**To:** Marcus Wei - Platform Engineer  
**From:** Integration Team  
**Subject:** Technical Design Review - API Integration Architecture  
**Version:** 1.0  
**Confidentiality:** Internal Use Only

---

## 1. Executive Summary

This document provides a technical design review of the integration architecture connecting SAP S/4HANA to Zetheta FinSight. The design implements an event-driven, resilient, and scalable integration pipeline that processes financial data with 4-hour SLA.

---

## 2. API Contract

### 2.1 Source API (SAP S/4HANA)

| Method | Endpoint | Description | Rate Limit | Batch Size |
|--------|----------|-------------|------------|------------|
| GET | /gl/journal-entries | GL Journal Entries | 100/min | 5,000 |
| GET | /ap/vendors | Vendor Master | 50/min | 5,000 |
| GET | /ap/open-items | AP Open Items | 100/min | 5,000 |
| GET | /ar/customers | Customer Master | 50/min | 5,000 |
| GET | /ar/open-items | AR Open Items | 100/min | 5,000 |
| GET | /cost-centres | Cost Centre Master | 50/min | 5,000 |
| GET | /profit-centres | Profit Centre Master | 50/min | 5,000 |
| GET | /budget/actuals | Budget vs Actual | 20/min | 5,000 |
| GET | /procurement/purchase-orders | Purchase Orders | 50/min | 5,000 |
| GET | /sales/orders | Sales Orders | 50/min | 5,000 |
| GET | /assets | Fixed Assets | 20/min | 5,000 |
| GET | /bank/statements | Bank Statements | 20/min | 5,000 |

### 2.2 Destination API (Zetheta FinSight)

| Method | Endpoint | Description | Rate Limit | Batch Size |
|--------|----------|-------------|------------|------------|
| POST | /journal-entries | Load GL Journal Entries | 1,000/min | 500 |
| POST | /vendors | Load Vendors | 1,000/min | 1,000 |
| POST | /customers | Load Customers | 1,000/min | 1,000 |
| POST | /cost-centres | Load Cost Centres | 500/min | 1,000 |
| POST | /profit-centres | Load Profit Centres | 500/min | 1,000 |
| POST | /purchase-orders | Load Purchase Orders | 500/min | 500 |
| POST | /sales-orders | Load Sales Orders | 500/min | 500 |
| POST | /assets | Load Assets | 200/min | 500 |
| POST | /bank-statements | Load Bank Statements | 200/min | 500 |
| POST | /materials | Load Materials | 500/min | 1,000 |
| POST | /budget-actual | Load Budget vs Actual | 500/min | 1,000 |
| POST | /reconciliation | Store Reconciliation Results | 50/min | N/A |

### 2.3 API Versioning Strategy
/api/v1/journal-entries → Current stable version
/api/v2/journal-entries → Future version (breaking changes)

Version Headers:
Accept: application/json;version=1.0

text

---

## 3. Authentication Flow

### 3.1 OAuth 2.0 Flow
┌─────────────┐ ┌─────────────┐
│ Integration │ │ Auth │
│ Pipeline │ │ Server │
└──────┬──────┘ └──────┬──────┘
│ │
│ 1. Client Credentials Grant │
│ POST /oauth2/token │
│ client_id, client_secret, scope │
│──────────────────────────────────────────────────▶│
│ │
│ 2. Access Token (JWT) │
│ access_token, token_type, expires_in │
│◀──────────────────────────────────────────────────│
│ │
│ 3. API Request with Bearer Token │
│ Authorization: Bearer <token> │
│──────────────────────────────────────────────────▶│
│ │
│ 4. Token Validated, Request Processed │
│◀──────────────────────────────────────────────────│
│ │
│ 5. Token Refresh (if expired) │
│ POST /oauth2/token │
│ refresh_token │
│──────────────────────────────────────────────────▶│
│ │
│ 6. New Access Token │
│◀──────────────────────────────────────────────────│

text

### 3.2 Token Configuration

| Parameter | Value |
|-----------|-------|
| **Token Type** | JWT (JSON Web Token) |
| **Algorithm** | RS256 (RSA) |
| **Expiry** | 1 hour |
| **Refresh** | Automatic before expiry |
| **Scopes** | read:gl, read:ap, read:ar, write:journal, write:vendor, write:customer |
| **Idempotency Retention** | 24 hours |

### 3.3 Authentication Implementation

```python
# Token manager implementation
class OAuth2TokenManager:
    """Manages OAuth2 token lifecycle."""
    
    def __init__(self, client_id, client_secret, token_url):
        self.client_id = client_id
        self.client_secret = client_secret
        self.token_url = token_url
        self.access_token = None
        self.expires_at = 0
    
    def get_token(self):
        """Get valid access token (refreshes if expired)."""
        if self._is_expired():
            self._refresh_token()
        return self.access_token
    
    def _is_expired(self):
        """Check if token is expired (with 5 min buffer)."""
        return time.time() + 300 >= self.expires_at
    
    def _refresh_token(self):
        """Refresh access token."""
        response = requests.post(
            self.token_url,
            data={
                'grant_type': 'client_credentials',
                'client_id': self.client_id,
                'client_secret': self.client_secret,
                'scope': 'write:journal write:vendor'
            }
        )
        data = response.json()
        self.access_token = data['access_token']
        self.expires_at = time.time() + data['expires_in']
4. Throughput Expectations
4.1 Daily Data Volumes
Domain	Daily Records	Peak/Min	Batch Size	Frequency
GL	70,000	2,000	500	30 min
AP	10,000	500	500	30 min
AR	8,000	400	500	30 min
Cost Centre	500	50	500	Daily
Profit Centre	200	20	500	Daily
Purchase Orders	5,000	300	500	30 min
Sales Orders	3,000	200	500	30 min
Fixed Assets	100	10	500	Monthly
Bank Statements	1,000	100	500	Real-time
Materials	15,000	500	500	Daily
Budget/Actual	20,000	500	500	Daily
Total	~133,000	~4,500	-	-
4.2 Peak Throughput Requirements
Metric	Value	Measurement
Peak Records/Min	4,500	Records processed per minute
Peak Records/Sec	75	Records processed per second
Concurrent Requests	500	Simultaneous API calls
Network Bandwidth	25 Mbps	Data transfer rate
API Calls/Min	1,000	API requests per minute
Throughput Sustained	37.8 rec/s	Average processing rate
4.3 SLA Requirements
SLA Metric	Target	Measurement
Data Freshness	< 4 hours	SAP to FinSight latency
Processing Time	< 2 hours	Batch processing duration
API Latency P95	< 5 seconds	API response time
Error Rate	< 1%	Data quality failures
System Uptime	99.5%	Service availability
5. Platform Limitations
5.1 FinSight API Constraints
Constraint	Value	Impact	Mitigation
Rate Limit	1,000/min	May delay large batches	Batch splitting, retry with backoff
Batch Size	500 records	Need to split large batches	Batch splitting logic
Payload Size	10 MB	Large batches require splitting	Payload monitoring
Timeout	30 seconds	Retry needed for slow operations	Exponential backoff retry
Concurrent Requests	50	Queue mechanism needed	Request queuing
Idempotency Keys	24 hour retention	Duplicate detection	Idempotency key management
5.2 SAP ODP Constraints
Constraint	Value	Impact	Mitigation
Extraction Frequency	Every 30 min	Real-time limitation	Acceptable for financial data
RFC Connections	50 max	Connection pooling needed	Connection manager with pooling
Batch Window	01:00-04:30	No extraction during window	Schedule avoidance
Record Size	10,000 per batch	Pagination needed	ODP pagination
Delta Token	Required	Tracking needed	Delta token management
5.3 Network Constraints
Constraint	Value	Impact	Mitigation
Bandwidth	450 Mbps	Throttling needed	Traffic shaping
Business Hours Usage	< 25%	Reduced throughput	Schedule large jobs off-hours
Latency	50-100ms	API response time	Connection pooling
VPN Availability	99.9%	Disaster recovery	Multiple VPN tunnels
6. Proposed Changes
6.1 New Endpoints Required
Endpoint	Method	Purpose	Priority
/reconciliation/status	GET	Get current reconciliation status	P1
/reconciliation/breaks	GET	Get details of reconciliation breaks	P1
/reconciliation/history	GET	Get reconciliation history	P2
/dlq/records	GET	Get list of DLQ records	P1
/dlq/records/{id}	GET	Get specific DLQ record	P1
/dlq/records/{id}/reprocess	POST	Reprocess DLQ record	P1
/dlq/records/{id}/resolve	POST	Resolve DLQ record	P1
/health	GET	Service health check	P1
/metrics	GET	Performance metrics	P2
/rate-limits	GET	Current rate limit status	P2
6.2 Schema Extensions
json
{
    "journalEntry": {
        // Current fields...
        "gstBreakdown": {
            "cgst": "number",
            "sgst": "number",
            "igst": "number",
            "taxCode": "string",
            "taxRate": "number"
        },
        "metadata": {
            "batchId": "string",
            "sourceSystem": "string",
            "processedAt": "datetime",
            "version": "string"
        },
        "reconciliation": {
            "sourceChecksum": "string",
            "targetChecksum": "string",
            "reconciledAt": "datetime",
            "status": "string"
        }
    }
}
6.3 Configuration Changes
yaml
# FinSight API Configuration
rate_limits:
  journal_entries: 1000
  vendors: 1000
  customers: 1000
  cost_centres: 500
  profit_centres: 500
  purchase_orders: 500
  sales_orders: 500
  assets: 200
  bank_statements: 200
  materials: 500
  budget_actual: 500
  reconciliation: 50

batch_sizes:
  default: 500
  journal_entries: 500
  vendors: 1000
  customers: 1000
  cost_centres: 1000
  profit_centres: 1000
  materials: 1000
  budget_actual: 1000

timeouts:
  default: 30
  journal_entries: 60
  vendors: 60
  customers: 60
  reconciliation: 120

retry_config:
  max_attempts: 3
  base_delay: 2
  max_delay: 60
  jitter_factor: 0.2
7. Engineering Support Requests
7.1 Platform Changes Required
Request	Priority	Impact	Timeline
Increase rate limit for journal entries to 2000/min	P2	Faster loading	1 week
Add reconciliation status endpoint	P1	Monitoring	Immediate
Add DLQ management endpoints	P1	Operations	Immediate
Increase batch size limit to 1000	P3	Efficiency	2 weeks
Add GST breakdown fields to schema	P1	Compliance	Immediate
Add metadata fields for tracking	P1	Audit	Immediate
Increase idempotency key retention to 48 hours	P2	Duplicate prevention	2 weeks
7.2 Infrastructure Requests
Request	Priority	Status	Timeline
AWS Mumbai region access (ap-south-1)	P1	✅ Approved	Immediate
VPN tunnel to Pune DC	P1	⏳ Pending	1 week
Security group rules for API Gateway	P1	⏳ Pending	Immediate
Load balancer configuration (AWS ALB)	P2	✅ Approved	2 weeks
Grafana dashboard permissions	P2	✅ Approved	Immediate
PagerDuty integration setup	P2	⏳ Pending	2 weeks
S3 bucket for DLQ storage	P1	✅ Approved	Immediate
RDS PostgreSQL instance	P1	✅ Approved	Immediate
7.3 Support Requirements
Request	Priority	Details
24/7 Monitoring	P1	PagerDuty on-call rotation
L1/L2 Support	P1	Support team training
Runbook Documentation	P1	Detailed runbook
SLA Definition	P1	Define SLAs with stakeholders
On-call Rotation	P2	Engineering on-call schedule
8. Design Decisions
8.1 Architecture Decisions
Decision	Rationale	Alternative	Why Chosen
OAuth 2.0	Industry standard, secure, flexible	API Key	Security and scalability
Kafka for decoupling	Scalability, replay capability, durability	Direct API, RabbitMQ	High throughput requirements
Idempotency	Prevent duplicate processing	Unique keys only	Safe retries
Circuit Breaker	Prevent cascade failures	Retry only	Service protection
Dead Letter Queue	Handle failures gracefully	Drop records	Data integrity
Event-driven	Real-time processing	Batch processing	4-hour SLA requirement
Microservices	Scalability, maintainability	Monolith	Team structure and scale
8.2 Technology Decisions
Component	Selection	Rationale
API Gateway	Kong	Open-source, flexible, rate limiting
Message Broker	Apache Kafka	High throughput, durability
Transformation Engine	Apache Spark	Distributed processing, scalability
Scheduler	Apache Airflow	DAG dependencies, monitoring
Reconciliation	Python + SQL	Flexibility, cost-effective
Monitoring	Prometheus + Grafana	Open-source, industry standard
Logging	ELK Stack	Open-source, scalable
Alerting	PagerDuty	Industry standard
Infrastructure	AWS Mumbai	RBI data localization
9. Performance Considerations
9.1 Bottlenecks Identified
Bottleneck	Impact	Mitigation
SAP RFC Pool (50 max)	Extraction delays	Connection pooling, reduce concurrent
FinSight Rate Limits	Loading delays	Batch splitting, backoff
Network Bandwidth (450 Mbps)	Transfer delays	Traffic shaping, compression
Database I/O	Performance	Indexing, query optimization
9.2 Scalability Approach
text
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SCALABILITY APPROACH                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Horizontal Scaling:                                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ • API Gateway: Multiple pods                                        │    │
│  │ • Transformation Engine: Worker pool                                │    │
│  │ • Kafka: Partition increase                                         │    │
│  │ • Consumers: Consumer group scaling                                 │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  Vertical Scaling:                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ • Database: RDS scaling                                              │    │
│  │ • Memory: JVM heap sizing                                            │    │
│  │ • CPU: Node scaling                                                 │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  Scaling Triggers:                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ • CPU > 70% → Scale out                                             │    │
│  │ • Memory > 80% → Scale up                                            │    │
│  │ • Queue > 10,000 → Scale out                                        │    │
│  │ • Error Rate > 5% → Investigate                                     │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
10. Review Checklist
Item	Status	Reviewer	Date
API Contract	___	___	___
- All endpoints defined	___	___	___
- Request/response schemas complete	___	___	___
- Error responses defined	___	___	___
Authentication Flow	___	___	___
- OAuth 2.0 configured	___	___	___
- Token refresh mechanism	___	___	___
- Scopes defined	___	___	___
Throughput Requirements	___	___	___
- Volume estimates validated	___	___	___
- Rate limits acceptable	___	___	___
- Batch sizes optimized	___	___	___
Platform Limitations	___	___	___
- All constraints documented	___	___	___
- Mitigations in place	___	___	___
- Workarounds identified	___	___	___
Security Review	___	___	___
- Authentication secure	___	___	___
- Data encryption in transit	___	___	___
- Data encryption at rest	___	___	___
- Audit logging configured	___	___	___
Performance Review	___	___	___
- Scalability tested	___	___	___
- Bottlenecks identified	___	___	___
- Optimization plan in place	___	___	___
Infrastructure Review	___	___	___
- All resources provisioned	___	___	___
- Security groups configured	___	___	___
- Monitoring enabled	___	___	___
- Backups configured	___	___	___
11. Sign-off
Role	Name	Signature	Date
Platform Engineer	Marcus Wei	__________	______
Integration Lead	_______________	__________	______
Architecture Lead	_______________	__________	______
Security Lead	_______________	__________	______
12. Next Steps
Action	Owner	Timeline
Address review comments	Integration Team	3 days
Update documentation	Integration Team	5 days
Finalize API specification	Platform Engineer	1 week
Provision infrastructure	DevOps	1 week
Begin development	Integration Team	After approval
Appendix A: Error Codes
Code	HTTP Status	Description	Action
INVALID_REQUEST	400	Request validation failed	Fix request
INVALID_DATE_FORMAT	400	Date format invalid	Use ISO 8601
TOKEN_EXPIRED	401	OAuth token expired	Refresh token
INVALID_TOKEN	401	Invalid token	Re-authenticate
INSUFFICIENT_SCOPE	403	Missing scope	Add scope
TENANT_LOCKED	403	Tenant in maintenance	Retry later
RESOURCE_NOT_FOUND	404	Resource not found	Check resource
DUPLICATE_ENTRY	409	Duplicate record	Check idempotency
RATE_LIMITED	429	Rate limit exceeded	Retry with backoff
INTERNAL_ERROR	500	Internal server error	Retry, contact support
SERVICE_UNAVAILABLE	503	Service unavailable	Retry later
Appendix B: Metrics
Metric	Name	Description
Throughput	records_processed_total	Total records processed
Latency	api_duration_seconds	API response time
Error Rate	errors_total	Total errors
DLQ Depth	kafka_consumer_group_lag	DLQ records
Reconciliation Score	reconciliation_score	Data quality score
Circuit Breaker State	circuit_breaker_state	CB state
Resource Usage	node_cpu_utilisation	CPU usage
Resource Usage	node_memory_utilisation	Memory usage
Appendix C: Dependencies
Dependency	Owner	Status	Impact
SAP ODP Subscriptions	SAP Basis	✅	Critical
FinSight API Availability	Platform	✅	Critical
AWS Infrastructure	DevOps	⏳	Critical
Network Connectivity	Network	⏳	Critical
Database	DBA	✅	High
Monitoring	DevOps	⏳	High