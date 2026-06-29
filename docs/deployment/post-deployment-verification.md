# Post-Deployment Verification

## Overview

This document defines 12+ post-deployment verification checks that must be completed within 1 hour of deployment. All checks must pass before the deployment is considered successful.

---

## Verification Timeline
┌─────────────────────────────────────────────────────────────────────────────┐
│ POST-DEPLOYMENT VERIFICATION TIMELINE │
├─────────────────────────────────────────────────────────────────────────────┤
│ │
│ T+0: Deployment Complete │
│ ┌─────────────────────────────────────────────────────────────────────┐ │
│ │ Immediate Checks (5 minutes) │ │
│ │ ✓ Service Health │ │
│ │ ✓ API Gateway │ │
│ │ ✓ Kafka Connectivity │ │
│ │ ✓ Database Connectivity │ │
│ └─────────────────────────────────────────────────────────────────────┘ │
│ ▼ │
│ T+5: Core Services (10 minutes) │
│ ┌─────────────────────────────────────────────────────────────────────┐ │
│ │ ✓ FinSight API │ │
│ │ ✓ SAP Connectivity │ │
│ │ ✓ Alerting │ │
│ │ ✓ Dashboards │ │
│ └─────────────────────────────────────────────────────────────────────┘ │
│ ▼ │
│ T+15: Data Flow (20 minutes) │
│ ┌─────────────────────────────────────────────────────────────────────┐ │
│ │ ✓ Data Flow │ │
│ │ ✓ Reconciliation │ │
│ │ ✓ DLQ │ │
│ │ ✓ Logs │ │
│ └─────────────────────────────────────────────────────────────────────┘ │
│ ▼ │
│ T+35: Final Sign-off (25 minutes) │
│ ┌─────────────────────────────────────────────────────────────────────┐ │
│ │ ✓ All Checks Passed │ │
│ │ ✓ Sign-off Complete │ │
│ │ ✓ Go-Live Notification │ │
│ └─────────────────────────────────────────────────────────────────────┘ │
│ │
│ Total Verification Time: 1 hour │
└─────────────────────────────────────────────────────────────────────────────┘

text

---

## Section 1: Immediate Checks (T+0 to T+5)

### Check 1: Service Health

| Attribute | Value |
|-----------|-------|
| **Check ID** | VER-001 |
| **Description** | All Kubernetes pods are running |
| **Type** | Service Health |
| **Time** | T+0 to T+1 |

**Command:**
```bash
kubectl get pods -n integration
Expected Result:

text
NAME                                     READY   STATUS    RESTARTS   AGE
api-gateway-7d8f9b6c5-abc12              1/1     Running   0          5m
transformation-engine-6c7d8e9f-xyz34     1/1     Running   0          5m
scheduler-5b6c7d8e-abcd5                 1/1     Running   0          5m
reconciliation-4a5b6c7d-efgh6            1/1     Running   0          5m
kafka-consumer-3a4b5c6d-ijkl7            1/1     Running   0          5m
Verification Command:

bash
kubectl get pods -n integration | grep -v Running
# Expected: No output (all pods Running)
Status	Pass/Fail
___	___
Check 2: API Gateway Health
Attribute	Value
Check ID	VER-002
Description	API Gateway is healthy and responding
Type	Service Health
Time	T+0 to T+2
Command:

bash
curl -s -o /dev/null -w "%{http_code}" https://api.integration.com/health
Expected Result: 200

Verification Command:

bash
# Full health check
curl -s https://api.integration.com/health | jq
Expected Output:

json
{
    "status": "healthy",
    "version": "1.0.0",
    "timestamp": "2026-03-29T02:00:00+05:30",
    "services": {
        "auth": "healthy",
        "router": "healthy",
        "rate_limiter": "healthy"
    }
}
Status	Pass/Fail
___	___
Check 3: Kafka Connectivity
Attribute	Value
Check ID	VER-003
Description	Kafka is reachable and all topics exist
Type	Service Health
Time	T+2 to T+4
Command:

bash
kafka-topics --bootstrap-server kafka:9092 --list
Expected Result:

text
GL-JournalEntries
AP-OpenItems
AR-OpenItems
CostCentre
ProfitCentre
PurchaseOrders
SalesOrders
FixedAssets
BankStatements
MaterialLedger
BudgetActual
dlq
Verification Command:

bash
# Check topic partitions
kafka-topics --bootstrap-server kafka:9092 --describe
Expected Output:

text
Topic: GL-JournalEntries    PartitionCount: 12    ReplicationFactor: 3
Topic: AP-OpenItems         PartitionCount: 8     ReplicationFactor: 3
Topic: AR-OpenItems         PartitionCount: 8     ReplicationFactor: 3
Status	Pass/Fail
___	___
Check 4: Database Connectivity
Attribute	Value
Check ID	VER-004
Description	PostgreSQL database is reachable
Type	Service Health
Time	T+3 to T+5
Command:

bash
psql -h postgres.integration.com -U dlq_user -d dlq_db -c "SELECT 1;"
Expected Result: 1

Verification Command:

bash
# Check database status
psql -h postgres.integration.com -U dlq_user -d dlq_db -c "SELECT version();"
Expected Output:

text
PostgreSQL 14.5 on x86_64-pc-linux-gnu
Status	Pass/Fail
___	___
Section 2: Core Services Check (T+5 to T+15)
Check 5: FinSight API Health
Attribute	Value
Check ID	VER-005
Description	FinSight API is healthy and responding
Type	Service Health
Time	T+5 to T+7
Command:

bash
curl -s -o /dev/null -w "%{http_code}" https://api.finsight.zetheta.com/health
Expected Result: 200

Verification Command:

bash
# Check FinSight API status
curl -s https://api.finsight.zetheta.com/health | jq
Expected Output:

json
{
    "status": "healthy",
    "version": "4.2.0",
    "timestamp": "2026-03-29T02:00:00+05:30",
    "services": {
        "database": "healthy",
        "cache": "healthy",
        "queue": "healthy"
    }
}
Status	Pass/Fail
___	___
Check 6: SAP Connectivity
Attribute	Value
Check ID	VER-006
Description	SAP system is reachable
Type	Service Health
Time	T+7 to T+10
Command:

bash
python3 -c "
import pyrfc
conn = pyrfc.Connection(
    ashost='sap-prod.meridian.co.in',
    sysnr='00',
    client='100',
    user='integration_user',
    passwd='password'
)
result = conn.call('RFC_PING')
print('SAP Connection: OK')
"
Expected Result:

text
SAP Connection: OK
Verification Command:

bash
# Check ODP subscription
python3 -c "
import pyrfc
conn = pyrfc.Connection(...)
result = conn.call('ODP_SUBSCRIPTION_CHECK', provider='Z_CDS_ACDOCA_DELTA')
print(f'ODP Status: {result}')
"
Status	Pass/Fail
___	___
Check 7: Alerting System
Attribute	Value
Check ID	VER-007
Description	Alerting system is configured and working
Type	Service Health
Time	T+10 to T+12
Command:

bash
curl -s http://prometheus:9090/api/v1/alerts | jq '.data.alerts[] | select(.state=="firing")'
Expected Result: No firing alerts

Verification Command:

bash
# Check Prometheus targets
curl -s http://prometheus:9090/api/v1/targets | jq '.data.activeTargets[].health'
Expected Result: All targets show "up"

Status	Pass/Fail
___	___
Check 8: Grafana Dashboards
Attribute	Value
Check ID	VER-008
Description	Grafana dashboards are accessible
Type	Service Health
Time	T+12 to T+15
Command:

bash
curl -s -o /dev/null -w "%{http_code}" https://grafana.integration.com/d/integration-pipeline
Expected Result: 200

Verification Command:

bash
# Check dashboard panels are loading
curl -s https://grafana.integration.com/api/dashboards/uid/integration-pipeline | jq '.dashboard.panels | length'
Expected Result: 12 (all 12 panels)

Status	Pass/Fail
___	___
Section 3: Data Flow Checks (T+15 to T+35)
Check 9: Data Flow Verification
Attribute	Value
Check ID	VER-009
Description	Data is flowing through the pipeline
Type	Data Flow
Time	T+15 to T+25
Command:

bash
# Trigger a test extraction
curl -X POST http://scheduler:8793/api/v1/jobs/GL/trigger \
  -H "Content-Type: application/json" \
  -d '{"test": true, "batch_size": 10}'
Expected Result: {"job_id": "JOB-20260329-0200", "status": "TRIGGERED"}

Verification Command:

bash
# Wait 2 minutes
sleep 120

# Check job status
curl -s http://scheduler:8793/api/v1/jobs/JOB-20260329-0200/status
Expected Output:

json
{
    "job_id": "JOB-20260329-0200",
    "status": "COMPLETED",
    "records_processed": 10,
    "records_failed": 0,
    "start_time": "2026-03-29T02:15:00+05:30",
    "end_time": "2026-03-29T02:15:45+05:30",
    "duration_seconds": 45
}
Status	Pass/Fail
___	___
Check 10: Reconciliation Verification
Attribute	Value
Check ID	VER-010
Description	Reconciliation is working correctly
Type	Data Flow
Time	T+25 to T+30
Command:

bash
# Run test reconciliation
curl -X POST http://reconciliation:8090/api/v1/reconcile \
  -H "Content-Type: application/json" \
  -d '{"batch_id": "JOB-20260329-0200"}'
Expected Result: {"status": "RECONCILED", "overall_score": 100.00}

Verification Command:

bash
# Get reconciliation status
curl -s http://reconciliation:8090/api/v1/reconciliation/status | jq
Expected Output:

json
{
    "status": "RECONCILED",
    "overall_score": 100.00,
    "completeness": 100.00,
    "accuracy": 100.00,
    "timeliness": 100.00,
    "consistency": 100.00,
    "dimensions": {
        "source_records": 10,
        "target_records": 10,
        "matching_records": 10,
        "variance": 0.00
    }
}
Status	Pass/Fail
___	___
Check 11: Dead Letter Queue
Attribute	Value
Check ID	VER-011
Description	DLQ is healthy with minimal records
Type	Data Flow
Time	T+30 to T+32
Command:

bash
curl -s http://prometheus:9090/api/v1/query?query=kafka_consumer_group_lag{topic=\"dlq\"}
Expected Result: < 100 records

Verification Command:

bash
# Check DLQ records in database
psql -h postgres.integration.com -U dlq_user -d dlq_db -c "
    SELECT domain, status, COUNT(*) 
    FROM dlq_records 
    WHERE created_at >= NOW() - INTERVAL '1 hour'
    GROUP BY domain, status;
"
Expected Result: Minimal records

Status	Pass/Fail
___	___
Check 12: Log Verification
Attribute	Value
Check ID	VER-012
Description	No error logs in the last hour
Type	Data Flow
Time	T+32 to T+35
Command:

bash
curl -s -X GET "http://elasticsearch:9200/integration-logs-*/_search?q=level:ERROR" \
  -H "Content-Type: application/json" | jq '.hits.total.value'
Expected Result: 0 (no errors)

Verification Command:

bash
# Check warning logs
curl -s -X GET "http://elasticsearch:9200/integration-logs-*/_search?q=level:WARNING" \
  -H "Content-Type: application/json" | jq '.hits.total.value'
Expected Result: < 10 (acceptable)

Status	Pass/Fail
___	___
Section 4: Performance Metrics (T+35 to T+50)
Check 13: Throughput Verification
Attribute	Value
Check ID	VER-013
Description	Throughput meets baseline
Type	Performance
Time	T+35 to T+40
Command:

bash
curl -s http://prometheus:9090/api/v1/query?query=rate(records_processed_total[5m])
Expected Result: > 300 rec/s

Verification:

bash
# Check if throughput is within acceptable range
THROUGHPUT=$(curl -s http://prometheus:9090/api/v1/query?query=rate(records_processed_total[5m]) | jq '.data.result[0].value[1]')
if (( $(echo "$THROUGHPUT > 300" | bc -l) )); then
    echo "✅ Throughput OK"
else
    echo "❌ Throughput below threshold"
fi
Status	Pass/Fail
___	___
Check 14: Latency Verification
Attribute	Value
Check ID	VER-014
Description	Latency meets SLA
Type	Performance
Time	T+40 to T+45
Command:

bash
curl -s http://prometheus:9090/api/v1/query?query=histogram_quantile(0.95,rate(api_duration_seconds_bucket[5m]))
Expected Result: < 5 seconds

Status	Pass/Fail
___	___
Section 5: Final Sign-off
Verification Summary
Check ID	Check Name	Status
VER-001	Service Health	___
VER-002	API Gateway	___
VER-003	Kafka Connectivity	___
VER-004	Database Connectivity	___
VER-005	FinSight API	___
VER-006	SAP Connectivity	___
VER-007	Alerting System	___
VER-008	Grafana Dashboards	___
VER-009	Data Flow	___
VER-010	Reconciliation	___
VER-011	Dead Letter Queue	___
VER-012	Log Verification	___
VER-013	Throughput	___
VER-014	Latency	___
Sign-off
Role	Name	Signature	Date
Integration Lead	_______________	__________	______
DevOps Lead	_______________	__________	______
QA Lead	_______________	__________	______
Stakeholder	_______________	__________	______
Final Status
Attribute	Value
Deployment Status	⬜ SUCCESS / ⬜ FAILED
Verification Status	⬜ PASSED / ⬜ FAILED
Checks Passed	___ / 14
Checks Failed	___ / 14
Comments	_______________
Appendix: Troubleshooting
Common Issues and Fixes
Issue	Likely Cause	Fix
Pods not starting	Resource constraints	Check node resources
API Gateway 503	Service not ready	Wait for rollout
Kafka connection fail	Broker down	Check broker status
Database connection fail	Credentials issue	Verify secrets
SAP connection fail	Network issue	Check VPN tunnel
No data flow	ODP subscription	Check ODP status
Reconciliation fail	Data mismatch	Run manual reconciliation
Escalation Contacts
Issue Type	Contact	Phone
Infrastructure	DevOps Lead	___
Application	Integration Lead	___
SAP	SAP Basis Admin	___
FinSight	Platform Engineer	___
Network	Network Team	___