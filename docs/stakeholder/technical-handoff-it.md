# Technical Handoff: SAP to FinSight Integration

**Date:** March 29, 2026  
**To:** Rajesh Venkataraman - VP IT Infrastructure  
**From:** Integration Team  
**Subject:** Technical Handoff Documentation for IT Operations  
**Version:** 1.0  
**Confidentiality:** Internal Use Only

---

## 1. Executive Summary

This document provides a comprehensive technical handoff for the IT operations team to support and maintain the SAP to FinSight integration pipeline. All infrastructure, configuration, and operational details are documented for seamless transition.

---

## 2. System Architecture Overview

### 2.1 High-Level Architecture
┌─────────────────────────────────────────────────────────────────────────────┐
│ SYSTEM ARCHITECTURE OVERVIEW │
├─────────────────────────────────────────────────────────────────────────────┤
│ │
│ On-Premise (Pune DC) Cloud (AWS Mumbai) │
│ │
│ ┌─────────────────────┐ ┌─────────────────────┐ │
│ │ SAP S/4HANA │ │ AWS VPC │ │
│ │ 10.10.20.10 │ │ 10.20.0.0/16 │ │
│ │ Port: 3300 │ │ │ │
│ └──────────┬──────────┘ │ ┌───────────────┐ │ │
│ │ │ │ API Gateway │ │ │
│ │ VPN Tunnel │ │ 10.20.10.10 │ │ │
│ │ (IPSec) │ │ Port: 443 │ │ │
│ ├───────────────────────────────┤ └───────────────┘ │ │
│ │ │ │ │
│ ┌──────────┴──────────┐ │ ┌───────────────┐ │ │
│ │ Firewall │ │ │ Kafka Broker │ │ │
│ │ Palo Alto │ │ │ 10.20.10.20 │ │ │
│ └─────────────────────┘ │ │ Port: 9092 │ │ │
│ │ └───────────────┘ │ │
│ │ │ │
│ │ ┌───────────────┐ │ │
│ │ │ PostgreSQL │ │ │
│ │ │ 10.20.30.20 │ │ │
│ │ │ Port: 5432 │ │ │
│ │ └───────────────┘ │ │
│ │ │ │
│ │ ┌───────────────┐ │ │
│ │ │ Snowflake │ │ │
│ │ │ (SaaS) │ │ │
│ │ └───────────────┘ │ │
│ └─────────────────────┘ │
│ │
└─────────────────────────────────────────────────────────────────────────────┘

text

### 2.2 Component List

| Component | Version | Purpose | Location |
|-----------|---------|---------|----------|
| SAP S/4HANA | 2023 FPS02 | Source ERP System | Pune DC |
| API Gateway | Kong 3.4 | API Management | AWS Mumbai |
| Kafka | 3.5.0 | Event Streaming | AWS Mumbai |
| Transformation Engine | Apache Spark 3.4 | Data Transformation | AWS Mumbai |
| Scheduler | Apache Airflow 2.7 | Job Orchestration | AWS Mumbai |
| Reconciliation Service | Custom Python 3.11 | Data Reconciliation | AWS Mumbai |
| PostgreSQL | 14.5 | Metadata Storage | AWS Mumbai |
| Snowflake | Enterprise | Data Warehouse | AWS Mumbai (SaaS) |
| Prometheus | 2.45 | Metrics Collection | AWS Mumbai |
| Grafana | 10.2 | Dashboard UI | AWS Mumbai |
| ELK Stack | 8.10 | Logging | AWS Mumbai |
| PagerDuty | Latest | Alerting | SaaS |

---

## 3. Changes to SAP Environment

### 3.1 New RFC Destinations

| RFC Destination | Purpose | Port | Protocol | Authentication |
|-----------------|---------|------|----------|----------------|
| FIN_SIGHT_API | FinSight API calls | 443 | HTTPS | OAuth 2.0 |
| KAFKA_PRODUCER | Kafka event streaming | 9092 | TCP | SSL Certificate |
| METADATA_SYNC | Master data extraction | 3300 | RFC | System User |
| ODP_EXTRACT | ODP delta extraction | 3300 | RFC | System User |
| TRANSFORM_ENGINE | Transformation service | 8080 | HTTP | Internal |

### 3.2 ODP Subscriptions

| Provider ID | CDS View | Frequency | Data Domain | Status |
|-------------|----------|-----------|-------------|--------|
| Z_CDS_ACDOCA_DELTA | I_JournalEntry | 30 min | GL | Active |
| Z_CDS_BSIK_DELTA | I_VendorOpenItem | 30 min | AP | Active |
| Z_CDS_BSID_DELTA | I_CustomerOpenItem | 30 min | AR | Active |
| Z_CDS_CSKS_DELTA | I_CostCentre | Daily | Cost | Active |
| Z_CDS_CEPC_DELTA | I_ProfitCentre | Daily | Profit | Active |
| Z_CDS_EKKO_DELTA | I_PurchaseOrder | 30 min | Procurement | Active |
| Z_CDS_VBAK_DELTA | I_SalesOrder | 30 min | Sales | Active |
| Z_CDS_MBEW_DELTA | I_Material | Daily | Materials | Active |
| Z_CDS_ANLA_DELTA | I_Asset | Monthly | Assets | Active |
| Z_CDS_FEBEP_DELTA | I_BankStatement | Real-time | Banking | Active |

### 3.3 Custom CDS Views

```sql
-- Z_CDS_ACDOCA_DELTA
@OData.publish: true
@Analytics.dataExtraction.enabled: true
@Analytics.delta.enabled: true
@ClientHandling.algorithm: #SESSION_VARIABLE
define view Z_CDS_ACDOCA_DELTA
  as select from ACDOCA
  association [0..1] to BKPF as _header
     on $projection.BUKRS = _header.BUKRS
    and $projection.BELNR = _header.BELNR
    and $projection.GJAHR = _header.GJAHR
  association [0..1] to SKAT as _glaccount
     on $projection.RACCT = _glaccount.SAKNR
{
  key BUKRS,
  key BELNR,
  key GJAHR,
  key BUZEI,
      BUDAT,
      BLDAT,
      RACCT,
      _glaccount.TXT50 as GL_ACCOUNT_NAME,
      HSL,
      WSL,
      RHCUR,
      RWCUR,
      KOSTL,
      PRCTR,
      MONAT,
      XBLNR,
      _header.BLART,
      _header.USNAM,
      _header.TCODE,
      _header.STBLG,
      _header.MONAT as HEADER_MONAT
}
where BUKRS in ('MC01', 'MC02', 'MC03')
  and (BUDAT >= '2024-01-01')
3.4 Transport Requests
Transport	Description	Type	Status	Date
TR-K-001	CDS Views for GL extraction	Workbench	Released	2026-03-15
TR-K-002	CDS Views for AP extraction	Workbench	Released	2026-03-15
TR-K-003	CDS Views for AR extraction	Workbench	Released	2026-03-15
TR-K-004	RFC Destinations	Workbench	Released	2026-03-18
TR-K-005	ODP Subscriptions	Workbench	Released	2026-03-20
TR-K-006	Authorization Objects	Workbench	Released	2026-03-20
TR-K-007	Service User Creation	Workbench	Released	2026-03-22
4. Network and Firewall Requirements
4.1 Network Architecture
On-Premise (Pune DC) Subnets
Subnet	IP Range	Purpose
DMZ	10.10.10.0/24	Firewall, VPN Gateway
Internal	10.10.20.0/24	SAP, MES, Active Directory
Database	10.10.30.0/24	Oracle, SQL Server
Cloud (AWS Mumbai) Subnets
Subnet	IP Range	Purpose	Availability Zone
Public	10.20.1.0/24	ALB, NAT Gateway	ap-south-1a
Private App	10.20.10.0/24	API Gateway, Kafka, Transform	ap-south-1a
Private Services	10.20.20.0/24	Scheduler, Reconciliation	ap-south-1a
Private DB	10.20.30.0/24	PostgreSQL, FinSight RDS	ap-south-1a
4.2 Firewall Rules
Inbound Rules (On-Premise → Cloud)
Rule ID	Source	Destination	Port	Protocol	Purpose
FW-001	Cloud (10.20.0.0/16)	SAP (10.10.20.10)	3300	RFC	ODP Extraction
FW-002	Cloud (10.20.0.0/16)	SAP (10.10.20.10)	3301	RFC	Master Data Sync
FW-003	Cloud (10.20.0.0/16)	SAP (10.10.20.11)	3300	RFC	QA System
FW-004	Cloud (10.20.0.0/16)	Active Directory	389	LDAP	User Authentication
Outbound Rules (On-Premise → Cloud)
Rule ID	Source	Destination	Port	Protocol	Purpose
FW-005	SAP (10.10.20.10)	Cloud (10.20.0.0/16)	9092	TCP	Kafka Publishing
FW-006	SAP (10.10.20.10)	Cloud (10.20.0.0/16)	443	HTTPS	FinSight API
FW-007	SAP (10.10.20.10)	Cloud (10.20.0.0/16)	5432	TCP	PostgreSQL
FW-008	SAP (10.10.20.10)	Cloud (10.20.0.0/16)	8080	HTTP	Transformation Engine
Cloud Security Groups
Security Group	Inbound Rules	Outbound Rules	Attached To
SG-APP	443 (HTTPS), 8080 (HTTP)	All	API Gateway, Transform
SG-KAFKA	9092 (TCP)	All	Kafka Brokers
SG-DB	5432 (PostgreSQL)	All	PostgreSQL RDS
SG-MGMT	22 (SSH), 3000 (Grafana)	All	Monitoring VMs
SG-VPN	500, 4500 (IPSec)	All	VPN Gateway
4.3 VPN Tunnel Specifications
Parameter	Value
Tunnel Type	IPSec VPN
Gateway	AWS VPN Gateway (vpg-xxxxxxxx)
Peer IP	203.0.113.1 (Pune DC)
Local Subnet	10.20.0.0/16
Remote Subnet	10.10.0.0/16
Encryption	AES-256
Authentication	Pre-Shared Key (PSK)
Phase 1	IKEv1, AES-256, SHA-256
Phase 2	ESP, AES-256, SHA-256
Dead Peer Detection	Enabled (10 seconds)
Redundancy	2 VPN tunnels (Active/Passive)
4.4 Load Balancer Configuration
text
┌─────────────────────────────────────────────────────────────────────────────┐
│                         LOAD BALANCER CONFIGURATION                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  AWS Application Load Balancer (ALB)                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ • Name: integration-alb                                             │    │
│  │ • Type: Application Load Balancer                                   │    │
│  │ • Scheme: Internet-facing                                           │    │
│  │ • Listeners:                                                        │    │
│  │   - HTTP:80 → Redirect to HTTPS                                    │    │
│  │   - HTTPS:443 → Forward to Target Groups                           │    │
│  │ • SSL Certificate: *.integration.meridian.co.in                    │    │
│  │ • Idle Timeout: 60 seconds                                         │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  Target Groups:                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ • Name: integration-api-gw                                           │    │
│  │ • Protocol: HTTP                                                    │    │
│  │ • Port: 8080                                                        │    │
│  │ • Health Check: /health                                             │    │
│  │ • Targets: API Gateway Pods                                         │    │
│  │ • Sticky Sessions: Disabled                                         │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
5. Security Requirements
5.1 SAP Authorizations
Authorization Object	Description	Required	Status
S_RFC	RFC Access	Yes	✅
S_TABU_DIS	Table Access	Yes	✅
S_ODP	ODP Access	Yes	✅
S_DEVELOP	ABAP Development	No	-
S_PROGRAM	Program Execution	No	-
S_ADMI_FCD	Administrative Functions	No	-
S_USER_GRP	User Groups	No	-
S_BTCH_JOB	Batch Jobs	Yes	✅
S_CTS_ADMI	Transport System	No	-
S_ALERT	Alert Management	Yes	✅
5.2 Service Account
yaml
# SAP Service Account Configuration
username: INTEGRATION_USER
password: ******** (Stored in AWS Secrets Manager)
roles:
  - SAP_ODP_USER
  - SAP_RFC_USER
  - SAP_READ_USER
  - SAP_MONITOR_USER
  - SAP_BATCH_USER

authorization_groups:
  - ODP_ACCESS
  - RFC_ACCESS
  - TABLE_ACCESS
  - CDS_ACCESS

permissions:
  - RFC_PING
  - ODP_SUBSCRIPTION
  - TABLE_READ: 
    - ACDOCA
    - BKPF
    - BSEG
    - BSIK
    - BSID
    - LFA1
    - LFC1
    - KNA1
    - KNC1
    - CSKS
    - CSKT
    - CEPC
    - CEPCT
    - EKKO
    - EKPO
    - VBAK
    - VBAP
    - ANLA
    - ANLZ
    - FEBEP
    - FEBKO
    - MBEW
    - MARD
    - COSP
    - COSS
    - TCURR
  - CDS_VIEW:
    - Z_CDS_ACDOCA_DELTA
    - Z_CDS_BSIK_DELTA
    - Z_CDS_BSID_DELTA
    - Z_CDS_CSKS_DELTA
    - Z_CDS_CEPC_DELTA
    - Z_CDS_EKKO_DELTA
    - Z_CDS_VBAK_DELTA
    - Z_CDS_MBEW_DELTA
    - Z_CDS_ANLA_DELTA
    - Z_CDS_FEBEP_DELTA
5.3 Certificate Requirements
Certificate	Issuer	Expiry	Location	Status
SAP SSL Certificate	Internal CA	2027-12-31	SAP Trust Manager	✅
FinSight API Certificate	DigiCert	2027-06-30	AWS ACM	✅
Kafka SSL Certificate	Internal CA	2027-12-31	Kafka Keystore	✅
API Gateway SSL Certificate	DigiCert	2027-06-30	AWS ACM	✅
VPN Certificate	Internal CA	2027-12-31	VPN Gateway	✅
PostgreSQL SSL Certificate	Internal CA	2027-12-31	AWS RDS	✅
5.4 Data Encryption
Data State	Method	Standard	Status
In-Transit	TLS 1.2+	All communications encrypted	✅
At-Rest (SAP)	AES-256	SAP database encryption	✅
At-Rest (Kafka)	AES-256	Kafka disk encryption	✅
At-Rest (PostgreSQL)	AES-256	RDS encryption	✅
At-Rest (S3)	AES-256	S3 Server-Side Encryption	✅
At-Rest (DLQ)	AES-256	S3 + PostgreSQL	✅
Backup	AES-256	Encrypted backups	✅
6. Monitoring and Alerting
6.1 Existing Monitoring Integration
yaml
# Nagios Integration Configuration
service:
  name: INTEGRATION_PIPELINE
  environment: production
  team: integration

checks:
  API_GATEWAY_HEALTH:
    type: http
    url: https://api.integration.com/health
    interval: 60
    threshold: 200

  KAFKA_CONNECTIVITY:
    type: tcp
    host: kafka.integration.com
    port: 9092
    interval: 60

  DATABASE_HEALTH:
    type: postgres
    host: postgres.integration.com
    port: 5432
    interval: 60

  SAP_ODP_STATUS:
    type: custom
    script: check_sap_odp.py
    interval: 300

  FIN_SIGHT_API:
    type: http
    url: https://api.finsight.zetheta.com/health
    interval: 60
    threshold: 200

  RECONCILIATION_STATUS:
    type: custom
    script: check_reconciliation.py
    interval: 300

thresholds:
  API_GATEWAY: 200
  KAFKA_LAG: 500
  DATABASE_CONN: 1
  RECONCILIATION_SCORE: 99.5
  DLQ_DEPTH: 100

alerts:
  PAGERDUTY_INTEGRATION: enabled
  EMAIL_ALERT: enabled
  SLACK_ALERT: enabled
6.2 Prometheus Metrics
yaml
# Prometheus Configuration
scrape_configs:
  - job_name: 'integration-api'
    scrape_interval: 15s
    metrics_path: '/metrics'
    static_configs:
      - targets: ['api-gateway:8080', 'transformation-engine:8080']

  - job_name: 'kafka'
    scrape_interval: 30s
    static_configs:
      - targets: ['kafka:9092']

  - job_name: 'postgresql'
    scrape_interval: 30s
    static_configs:
      - targets: ['postgres-exporter:9187']

  - job_name: 'kubernetes'
    scrape_interval: 15s
    kubernetes_sd_configs:
      - role: pod

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']
6.3 Grafana Dashboards
Dashboard	UID	Purpose	URL
Pipeline Overview	integration-pipeline	Overall system health	/d/integration-pipeline
Performance Metrics	integration-performance	Throughput, latency	/d/integration-performance
Reconciliation	integration-reconciliation	Reconciliation status	/d/integration-reconciliation
SAP Health	integration-sap	SAP system health	/d/integration-sap
FinSight API	integration-finsight	API health	/d/integration-finsight
DLQ Management	integration-dlq	Dead Letter Queue	/d/integration-dlq
Resource Usage	integration-resources	CPU, memory, disk	/d/integration-resources
Kafka Monitoring	integration-kafka	Kafka metrics	/d/integration-kafka
7. Support and Operations
7.1 Support Model
text
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SUPPORT MODEL                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Level 1 - Support Team                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ • Monitoring alerts                                                  │    │
│  │ • Basic troubleshooting                                              │    │
│  │ • User issue resolution                                              │    │
│  │ • Escalation to L2 if needed                                         │    │
│  │ • Response Time: 15 minutes                                          │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                              ▼                                              │
│  Level 2 - Integration Team                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ • Error investigation                                                │    │
│  │ • DLQ review and resolution                                          │    │
│  │ • Performance issues                                                 │    │
│  │ • Configuration changes                                              │    │
│  │ • Response Time: 30 minutes                                          │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                              ▼                                              │
│  Level 3 - Platform Engineering                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ • Code fixes                                                         │    │
│  │ • Architectural changes                                              │    │
│  │ • Platform-level issues                                              │    │
│  │ • Security incidents                                                 │    │
│  │ • Response Time: 1 hour                                              │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                              ▼                                              │
│  Level 4 - Vendor Support                                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ • SAP support (Priya Deshmukh)                                       │    │
│  │ • Zetheta support (Marcus Wei)                                       │    │
│  │ • AWS support                                                        │    │
│  │ • Response Time: 2 hours                                             │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
7.2 Escalation Procedures
Level	Contact	Response Time	Escalation Time
L1	Support Team	15 minutes	30 minutes
L2	Integration Lead	30 minutes	1 hour
L3	Platform Engineer	1 hour	2 hours
L4	VP IT Infrastructure	2 hours	4 hours
7.3 On-Call Rotation Schedule
yaml
oncall_rotation:
  week1:
    primary: integration-lead-1
    secondary: integration-lead-2
    escalation: platform-engineer-1
  week2:
    primary: integration-lead-2
    secondary: integration-lead-3
    escalation: platform-engineer-2
  week3:
    primary: integration-lead-3
    secondary: integration-lead-1
    escalation: platform-engineer-1
  week4:
    primary: integration-lead-4
    secondary: integration-lead-2
    escalation: platform-engineer-2
7.4 Maintenance Schedule
System	Schedule	Duration	Impact	Window
SAP S/4HANA	2nd Saturday	8 hours	No extraction	22:00-06:00
FinSight	1st Sunday	4 hours	No loading	02:00-06:00
Kafka	3rd Saturday	4 hours	No streaming	02:00-06:00
PostgreSQL	3rd Saturday	2 hours	Read-only	03:00-05:00
API Gateway	3rd Saturday	1 hour	Downtime	02:00-03:00
Monitoring	Weekly	30 min	No impact	04:00-04:30
8. Handoff Checklist
8.1 Documentation Handoff
Document	Owner	Status	Date
Architecture Diagrams	Integration Team	✅	2026-03-29
API Specifications	Integration Team	✅	2026-03-29
Runbook	Integration Team	✅	2026-03-29
Troubleshooting Guide	Integration Team	✅	2026-03-29
Monitoring Dashboards	DevOps	✅	2026-03-29
Incident Response Plan	DevOps	✅	2026-03-29
Disaster Recovery Plan	IT Team	⏳	2026-04-05
Access Documentation	IT Team	⏳	2026-04-05
8.2 Access Handoff
Access	User	Owner	Status
SAP System	INTEGRATION_USER	SAP Basis	✅
AWS Console	Integration Team	DevOps	✅
Kafka	integration-consumer	DevOps	✅
PostgreSQL	dlq_user	DBA	✅
FinSight API	integration-client	Platform	✅
PagerDuty	integration-team	DevOps	✅
Grafana	integration-team	DevOps	✅
Git Repository	@ZethetaIntern	DevOps	✅
8.3 Training Handoff
Topic	Audience	Owner	Status
Architecture Overview	Support Team	Integration Team	✅
Monitoring Dashboards	Support Team	DevOps	✅
DLQ Management	Support Team	Integration Team	✅
Incident Response	Support Team	DevOps	✅
Troubleshooting	Support Team	Integration Team	✅
Rollback Procedure	All Teams	Integration Team	✅
9. Contact Information
9.1 Primary Contacts
Role	Name	Phone	Email
Integration Lead	_______________	+91 98765 43210	lead@integration.com
Project Manager	_______________	+91 98765 43211	pm@integration.com
SAP Basis Admin	Priya Deshmukh	+91 98765 43212	priya.deshmukh@meridian.co.in
Platform Engineer	Marcus Wei	+91 98765 43213	marcus.wei@zetheta.com
DevOps Lead	_______________	+91 98765 43214	devops@integration.com
DBA	_______________	+91 98765 43215	dba@meridian.co.in
Network Admin	_______________	+91 98765 43216	network@meridian.co.in
Security Lead	_______________	+91 98765 43217	security@meridian.co.in
Support Lead	_______________	+91 98765 43218	support@integration.com
9.2 Escalation Contacts
Level	Role	Name	Phone	Email
L1	Support Lead	_______________	+91 98765 43218	support@integration.com
L2	Integration Lead	_______________	+91 98765 43210	lead@integration.com
L3	VP IT Infrastructure	Rajesh Venkataraman	+91 98765 43219	rajesh.venkataraman@meridian.co.in
L4	CFO	Ananya Krishnan	+91 98765 43220	ananya.krishnan@meridian.co.in
10. Appendix
10.1 Common Issues and Fixes
Issue	Likely Cause	Fix	Owner
ODP extraction failing	ODP subscription inactive	Check ODP monitor (RSA1)	SAP Basis
Kafka lag increasing	Consumer not keeping up	Increase consumer group	DevOps
API rate limited	Too many requests	Implement backoff	Integration
Database connection failed	Connection pool exhausted	Increase pool size	DBA
Reconciliation break	Data mismatch	Investigate and reconcile	Support
Circuit breaker open	Service failure	Check service health	Engineering
10.2 Useful Commands
bash
# Kubernetes
kubectl get pods -n integration
kubectl logs -f deployment/api-gateway -n integration
kubectl describe pod <pod-name> -n integration

# Kafka
kafka-topics --bootstrap-server kafka:9092 --list
kafka-consumer-groups --bootstrap-server kafka:9092 --describe --group integration-consumer

# PostgreSQL
psql -h postgres.integration.com -U dlq_user -d dlq_db -c "SELECT COUNT(*) FROM dlq_records;"

# API
curl -s https://api.integration.com/health
curl -s http://scheduler:8793/api/v1/jobs/status

# Metrics
curl -s http://prometheus:9090/api/v1/query?query=up
11. Sign-off
Role	Name	Signature	Date
Integration Lead	_______________	__________	______
VP IT Infrastructure	Rajesh Venkataraman	__________	______
DevOps Lead	_______________	__________	______
SAP Basis Admin	Priya Deshmukh	__________	______
Support Lead	_______________	__________	______
