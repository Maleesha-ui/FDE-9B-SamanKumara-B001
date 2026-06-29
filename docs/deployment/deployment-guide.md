# Step-by-Step Deployment Guide

## Overview

This document provides the step-by-step deployment procedure for the SAP to FinSight integration pipeline. All steps must be executed in order and verified before proceeding to the next phase.

---

## Deployment Phases Overview
┌─────────────────────────────────────────────────────────────────────────────┐
│ DEPLOYMENT PHASES │
├─────────────────────────────────────────────────────────────────────────────┤
│ │
│ Phase 1: Pre-Deployment (30 minutes) │
│ ┌─────────────────────────────────────────────────────────────────────┐ │
│ │ 1.1 Notify Stakeholders │ │
│ │ 1.2 Backup Current Configuration │ │
│ │ 1.3 Run Pre-Deployment Checklist │ │
│ │ 1.4 Prepare Deployment Environment │ │
│ └─────────────────────────────────────────────────────────────────────┘ │
│ ▼ │
│ Phase 2: Infrastructure (15 minutes) │
│ ┌─────────────────────────────────────────────────────────────────────┐ │
│ │ 2.1 Update Kubernetes Configurations │ │
│ │ 2.2 Deploy Database Migrations │ │
│ │ 2.3 Deploy Kafka Topic Updates │ │
│ │ 2.4 Update Network Configuration │ │
│ └─────────────────────────────────────────────────────────────────────┘ │
│ ▼ │
│ Phase 3: Application (20 minutes) │
│ ┌─────────────────────────────────────────────────────────────────────┐ │
│ │ 3.1 Deploy API Gateway Updates │ │
│ │ 3.2 Deploy Transformation Engine │ │
│ │ 3.3 Deploy Scheduler Updates │ │
│ │ 3.4 Deploy Reconciliation Service │ │
│ │ 3.5 Deploy Monitoring Stack Updates │ │
│ └─────────────────────────────────────────────────────────────────────┘ │
│ ▼ │
│ Phase 4: Post-Deployment (30 minutes) │
│ ┌─────────────────────────────────────────────────────────────────────┐ │
│ │ 4.1 Run Smoke Tests │ │
│ │ 4.2 Verify Reconciliation │ │
│ │ 4.3 Monitor Alerts │ │
│ │ 4.4 Update Documentation │ │
│ └─────────────────────────────────────────────────────────────────────┘ │
│ ▼ │
│ Phase 5: Go-Live (10 minutes) │
│ ┌─────────────────────────────────────────────────────────────────────┐ │
│ │ 5.1 Enable Production Traffic │ │
│ │ 5.2 Verify End-to-End Flow │ │
│ │ 5.3 Send Success Notification │ │
│ └─────────────────────────────────────────────────────────────────────┘ │
│ │
│ Total Estimated Time: 1 hour 45 minutes │
└─────────────────────────────────────────────────────────────────────────────┘

text

---

## Phase 1: Pre-Deployment (30 minutes)

### Step 1.1: Notify Stakeholders (5 minutes)

#### Slack Notification
```bash
# Send deployment notification to Slack
curl -X POST https://hooks.slack.com/services/<TEAM_ID>/<CHANNEL_ID>/<WEBHOOK_SECRET>
  -H "Content-Type: application/json" \
  -d '{
    "text": "🚀 Integration Pipeline Deployment Starting",
    "blocks": [
      {
        "type": "header",
        "text": {
          "type": "plain_text",
          "text": "🚀 Integration Pipeline Deployment Starting"
        }
      },
      {
        "type": "section",
        "fields": [
          {
            "type": "mrkdwn",
            "text": "*Environment:*\nProduction"
          },
          {
            "type": "mrkdwn",
            "text": "*Start Time:*\n2026-03-29 02:00 IST"
          },
          {
            "type": "mrkdwn",
            "text": "*Expected Duration:*\n2 hours"
          },
          {
            "type": "mrkdwn",
            "text": "*Impact:*\nNone (Blue-Green Deployment)"
          }
        ]
      },
      {
        "type": "section",
        "text": {
          "type": "mrkdwn",
          "text": "*Deployment Details:*\n- Version: v1.2.0\n- Changes: Performance improvements, bug fixes\n- Rollback: Available within 15 minutes"
        }
      }
    ]
  }'
Email Notification
bash
# Send email notification
echo "Subject: [Integration] Deployment Starting - 2026-03-29 02:00 IST

Dear Team,

Integration Pipeline Deployment is starting at 02:00 IST.

Environment: Production
Version: v1.2.0
Expected Duration: 2 hours
Impact: None (Blue-Green Deployment)

Deployment Lead: Integration Team
Rollback Available: Yes (within 15 minutes)

Please monitor your emails for updates.

- Integration Team" | sendmail stakeholders@meridian.co.in
Step 1.2: Backup Current Configuration (10 minutes)
bash
# Create backup directory
BACKUP_DIR=/backup/deployment-$(date +%Y%m%d)
mkdir -p $BACKUP_DIR

# Backup Kubernetes configurations
echo "Backing up Kubernetes configurations..."
kubectl get all -n integration -o yaml > $BACKUP_DIR/kubernetes-$(date +%Y%m%d-%H%M%S).yaml
kubectl get configmaps -n integration -o yaml > $BACKUP_DIR/configmaps-$(date +%Y%m%d-%H%M%S).yaml
kubectl get secrets -n integration -o yaml > $BACKUP_DIR/secrets-$(date +%Y%m%d-%H%M%S).yaml

# Backup database
echo "Backing up database..."
pg_dump -h postgres.integration.com -U dlq_user -d dlq_db \
  --format=custom \
  --file=$BACKUP_DIR/postgres-$(date +%Y%m%d-%H%M%S).backup

# Backup Kafka topics
echo "Backing up Kafka topics..."
kafka-topics --bootstrap-server kafka:9092 --list > $BACKUP_DIR/kafka-topics-$(date +%Y%m%d-%H%M%S).txt

# Backup Kafka topic configurations
kafka-topics --bootstrap-server kafka:9092 --describe > $BACKUP_DIR/kafka-topics-desc-$(date +%Y%m%d-%H%M%S).txt

# Backup Grafana dashboards
echo "Backing up Grafana dashboards..."
curl -s -H "Authorization: Bearer $GRAFANA_API_KEY" \
  https://grafana.integration.com/api/search?type=dash-db \
  | jq -r '.[].uid' | while read uid; do
    curl -s -H "Authorization: Bearer $GRAFANA_API_KEY" \
      https://grafana.integration.com/api/dashboards/uid/$uid \
      > $BACKUP_DIR/grafana-$(date +%Y%m%d)-$uid.json
  done

# Backup application configuration
echo "Backing up application configuration..."
cp -r /etc/integration/ $BACKUP_DIR/config/

# Verify backups
echo "Verifying backups..."
ls -la $BACKUP_DIR/
Step 1.3: Run Pre-Deployment Checklist (10 minutes)
bash
# Run pre-deployment checklist verification
echo "Running pre-deployment checklist verification..."

# Function to check status
check_status() {
    if [ $? -eq 0 ]; then
        echo "✅ $1 - PASSED"
    else
        echo "❌ $1 - FAILED"
        exit 1
    fi
}

# Check 1: All pods running
kubectl get pods -n integration | grep -v Running > /dev/null
check_status "All pods running"

# Check 2: API Gateway healthy
curl -s -o /dev/null -w "%{http_code}" https://api.integration.com/health | grep 200 > /dev/null
check_status "API Gateway healthy"

# Check 3: Database connectivity
psql -h postgres.integration.com -U dlq_user -d dlq_db -c "SELECT 1;" > /dev/null
check_status "Database connectivity"

# Check 4: Kafka connectivity
kafka-topics --bootstrap-server kafka:9092 --list > /dev/null
check_status "Kafka connectivity"

# Check 5: FinSight API
curl -s -o /dev/null -w "%{http_code}" https://api.finsight.zetheta.com/health | grep 200 > /dev/null
check_status "FinSight API healthy"

# Check 6: SAP connectivity
python3 -c "import pyrfc; conn = pyrfc.Connection(ashost='sap-prod.meridian.co.in', sysnr='00', client='100', user='integration_user', passwd='password'); conn.call('RFC_PING')" > /dev/null
check_status "SAP connectivity"

echo "✅ All pre-deployment checks passed!"
Step 1.4: Prepare Deployment Environment (5 minutes)
bash
# Set environment variables
export ENVIRONMENT=production
export DEPLOYMENT_VERSION=v1.2.0
export NAMESPACE=integration
export DEPLOYMENT_TIME=$(date +%Y-%m-%d_%H-%M-%S)

# Create deployment tracking file
cat > /tmp/deployment-tracking.txt << EOF
DEPLOYMENT_ID: DEP-$(date +%Y%m%d-%H%M%S)
ENVIRONMENT: production
VERSION: v1.2.0
START_TIME: $(date)
STATUS: in_progress
DEPLOYED_BY: $(whoami)
EOF

# Verify deployment tracking
cat /tmp/deployment-tracking.txt
Phase 2: Infrastructure Deployment (15 minutes)
Step 2.1: Update Kubernetes Configurations (5 minutes)
bash
echo "Applying Kubernetes configurations..."

# Apply namespace
kubectl apply -f k8s/namespace.yaml

# Apply ConfigMaps
kubectl apply -f k8s/configmap.yaml

# Apply Secrets
kubectl apply -f k8s/secrets.yaml

# Apply Persistent Volumes
kubectl apply -f k8s/persistent-volumes.yaml

# Verify configurations
kubectl get configmaps -n integration
kubectl get secrets -n integration
kubectl get pv -n integration
Step 2.2: Deploy Database Migrations (5 minutes)
sql
-- Database migration script
-- migrations/001_add_reprocess_columns.sql

-- Start transaction
BEGIN;

-- Add new columns for reprocessing
ALTER TABLE dlq_records ADD COLUMN IF NOT EXISTS reprocess_count INTEGER DEFAULT 0;
ALTER TABLE dlq_records ADD COLUMN IF NOT EXISTS last_reprocess_at TIMESTAMP;
ALTER TABLE dlq_records ADD COLUMN IF NOT EXISTS reprocess_error TEXT;

-- Add new columns for audit
ALTER TABLE dlq_records ADD COLUMN IF NOT EXISTS reviewed_by VARCHAR(100);
ALTER TABLE dlq_records ADD COLUMN IF NOT EXISTS reviewed_at TIMESTAMP;
ALTER TABLE dlq_records ADD COLUMN IF NOT EXISTS resolution_notes TEXT;

-- Create new indexes
CREATE INDEX IF NOT EXISTS idx_dlq_reprocess_count ON dlq_records(reprocess_count);
CREATE INDEX IF NOT EXISTS idx_dlq_last_reprocess ON dlq_records(last_reprocess_at);
CREATE INDEX IF NOT EXISTS idx_dlq_reviewed_at ON dlq_records(reviewed_at);

-- Create new status history table
CREATE TABLE IF NOT EXISTS dlq_status_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    record_id UUID REFERENCES dlq_records(id),
    status VARCHAR(20) NOT NULL,
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    reason TEXT,
    changed_by VARCHAR(100)
);

CREATE INDEX IF NOT EXISTS idx_status_history_record ON dlq_status_history(record_id);
CREATE INDEX IF NOT EXISTS idx_status_history_changed_at ON dlq_status_history(changed_at DESC);

-- Create audit log table
CREATE TABLE IF NOT EXISTS deployment_audit (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deployment_id VARCHAR(50) NOT NULL,
    action VARCHAR(50) NOT NULL,
    action_by VARCHAR(100) NOT NULL,
    action_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    details JSONB,
    status VARCHAR(20) DEFAULT 'SUCCESS'
);

-- Verify migrations
SELECT 
    table_name, 
    column_name, 
    data_type 
FROM information_schema.columns 
WHERE table_name IN ('dlq_records', 'dlq_status_history', 'deployment_audit')
ORDER BY table_name, column_name;

-- Commit transaction
COMMIT;
bash
# Run database migrations
echo "Running database migrations..."

psql -h postgres.integration.com -U dlq_user -d dlq_db -f migrations/001_add_reprocess_columns.sql

if [ $? -eq 0 ]; then
    echo "✅ Database migrations completed successfully"
else
    echo "❌ Database migrations failed"
    exit 1
fi
Step 2.3: Deploy Kafka Topic Updates (3 minutes)
bash
echo "Deploying Kafka topic updates..."

# Create/update Kafka topics with correct partitions
TOPICS=(
    "GL-JournalEntries:12"
    "AP-OpenItems:8"
    "AR-OpenItems:8"
    "CostCentre:6"
    "ProfitCentre:6"
    "PurchaseOrders:8"
    "SalesOrders:6"
    "FixedAssets:4"
    "BankStatements:4"
    "MaterialLedger:6"
    "BudgetActual:6"
    "dlq:4"
)

for topic_config in "${TOPICS[@]}"; do
    topic=$(echo $topic_config | cut -d: -f1)
    partitions=$(echo $topic_config | cut -d: -f2)
    
    echo "Creating/updating topic: $topic (partitions: $partitions)"
    
    kafka-topics --bootstrap-server kafka:9092 \
        --create --if-not-exists \
        --topic "$topic" \
        --partitions "$partitions" \
        --replication-factor 3
done

# Verify topics
echo "Verifying topics..."
kafka-topics --bootstrap-server kafka:9092 --list
Step 2.4: Update Network Configuration (2 minutes)
bash
echo "Updating network configuration..."

# Update load balancer configuration
kubectl apply -f k8s/ingress.yaml

# Update network policies
kubectl apply -f k8s/network-policies.yaml

# Verify ingress
kubectl get ingress -n integration

# Test network connectivity
kubectl run -it --rm test-pod --image=busybox --restart=Never -- \
    wget -O- https://api.integration.com/health
Phase 3: Application Deployment (20 minutes)
Step 3.1: Deploy API Gateway Updates (5 minutes)
bash
echo "Deploying API Gateway updates..."

# Apply deployment
kubectl apply -f k8s/api-gateway-deployment.yaml

# Wait for rollout
kubectl rollout status deployment/api-gateway -n integration --timeout=5m

if [ $? -eq 0 ]; then
    echo "✅ API Gateway deployment successful"
else
    echo "❌ API Gateway deployment failed"
    exit 1
fi

# Verify pods
kubectl get pods -n integration | grep api-gateway

# Test API Gateway
curl -s -o /dev/null -w "%{http_code}" https://api.integration.com/health | grep 200
Step 3.2: Deploy Transformation Engine (5 minutes)
bash
echo "Deploying Transformation Engine..."

# Apply deployment
kubectl apply -f k8s/transformation-engine-deployment.yaml

# Wait for rollout
kubectl rollout status deployment/transformation-engine -n integration --timeout=5m

if [ $? -eq 0 ]; then
    echo "✅ Transformation Engine deployment successful"
else
    echo "❌ Transformation Engine deployment failed"
    exit 1
fi

# Verify pods
kubectl get pods -n integration | grep transformation-engine

# Test Transformation Engine
curl -s http://transformation-engine:8080/health | jq
Step 3.3: Deploy Scheduler Updates (3 minutes)
bash
echo "Deploying Scheduler updates..."

# Apply deployment
kubectl apply -f k8s/scheduler-deployment.yaml

# Wait for rollout
kubectl rollout status deployment/scheduler -n integration --timeout=3m

if [ $? -eq 0 ]; then
    echo "✅ Scheduler deployment successful"
else
    echo "❌ Scheduler deployment failed"
    exit 1
fi

# Verify pods
kubectl get pods -n integration | grep scheduler

# Test Scheduler
curl -s http://scheduler:8793/health
Step 3.4: Deploy Reconciliation Service (3 minutes)
bash
echo "Deploying Reconciliation Service..."

# Apply deployment
kubectl apply -f k8s/reconciliation-deployment.yaml

# Wait for rollout
kubectl rollout status deployment/reconciliation -n integration --timeout=3m

if [ $? -eq 0 ]; then
    echo "✅ Reconciliation Service deployment successful"
else
    echo "❌ Reconciliation Service deployment failed"
    exit 1
fi

# Verify pods
kubectl get pods -n integration | grep reconciliation

# Test Reconciliation Service
curl -s http://reconciliation:8090/health
Step 3.5: Deploy Monitoring Stack Updates (4 minutes)
bash
echo "Deploying Monitoring Stack updates..."

# Apply Prometheus configuration
kubectl apply -f k8s/prometheus-config.yaml

# Apply Grafana configuration
kubectl apply -f k8s/grafana-config.yaml

# Apply Alertmanager configuration
kubectl apply -f k8s/alertmanager-config.yaml

# Restart monitoring services
kubectl rollout restart deployment/prometheus -n monitoring
kubectl rollout restart deployment/grafana -n monitoring
kubectl rollout restart deployment/alertmanager -n monitoring

# Wait for restarts
kubectl rollout status deployment/prometheus -n monitoring
kubectl rollout status deployment/grafana -n monitoring
kubectl rollout status deployment/alertmanager -n monitoring

# Verify monitoring
curl -s http://prometheus:9090/-/healthy
curl -s http://grafana:3000/api/health
Phase 4: Post-Deployment Verification (30 minutes)
Step 4.1: Run Smoke Tests (10 minutes)
bash
echo "Running smoke tests..."

# Test script
cat > /tmp/smoke-tests.sh << 'EOF'
#!/bin/bash

FAILED=0

echo "Running smoke tests..."

# Test 1: API Gateway
echo -n "Test 1: API Gateway... "
curl -s -o /dev/null -w "%{http_code}" https://api.integration.com/health | grep 200 > /dev/null
if [ $? -eq 0 ]; then
    echo "✅ PASS"
else
    echo "❌ FAIL"
    FAILED=$((FAILED+1))
fi

# Test 2: Kafka
echo -n "Test 2: Kafka Connectivity... "
kafka-topics --bootstrap-server kafka:9092 --list > /dev/null
if [ $? -eq 0 ]; then
    echo "✅ PASS"
else
    echo "❌ FAIL"
    FAILED=$((FAILED+1))
fi

# Test 3: Database
echo -n "Test 3: Database Connectivity... "
psql -h postgres.integration.com -U dlq_user -d dlq_db -c "SELECT 1;" > /dev/null
if [ $? -eq 0 ]; then
    echo "✅ PASS"
else
    echo "❌ FAIL"
    FAILED=$((FAILED+1))
fi

# Test 4: FinSight API
echo -n "Test 4: FinSight API... "
curl -s -o /dev/null -w "%{http_code}" https://api.finsight.zetheta.com/health | grep 200 > /dev/null
if [ $? -eq 0 ]; then
    echo "✅ PASS"
else
    echo "❌ FAIL"
    FAILED=$((FAILED+1))
fi

# Test 5: SAP ODP
echo -n "Test 5: SAP ODP Connectivity... "
python3 -c "import pyrfc; conn = pyrfc.Connection(ashost='sap-prod.meridian.co.in', sysnr='00', client='100', user='integration_user', passwd='password'); conn.call('ODP_SUBSCRIPTION_CHECK', provider='Z_CDS_ACDOCA_DELTA')" > /dev/null
if [ $? -eq 0 ]; then
    echo "✅ PASS"
else
    echo "❌ FAIL"
    FAILED=$((FAILED+1))
fi

if [ $FAILED -eq 0 ]; then
    echo "✅ All smoke tests PASSED!"
    exit 0
else
    echo "❌ $FAILED smoke tests FAILED!"
    exit 1
fi
EOF

chmod +x /tmp/smoke-tests.sh
/tmp/smoke-tests.sh
Step 4.2: Verify Reconciliation (10 minutes)
bash
echo "Verifying reconciliation..."

# Run test reconciliation
curl -X POST http://reconciliation:8090/api/v1/reconcile/test \
  -H "Content-Type: application/json" \
  -d '{
    "batch_id": "BATCH-GL-20260329-0200",
    "test_data": true
  }'

# Check reconciliation status
sleep 30

curl -s http://reconciliation:8090/api/v1/reconciliation/status | jq

# Expected output:
# {
#   "status": "RECONCILED",
#   "overall_score": 100.00,
#   "completeness": 100.00,
#   "accuracy": 100.00,
#   "timeliness": 100.00,
#   "consistency": 100.00
# }
Step 4.3: Monitor Alerts (5 minutes)
bash
echo "Monitoring alerts..."

# Check active alerts
curl -s http://prometheus:9090/api/v1/alerts | jq '.data.alerts[] | select(.state=="firing")'

# Expected: No firing alerts

# Check alert history
curl -s http://prometheus:9090/api/v1/alerts | jq '.data.alerts[].state'
Step 4.4: Update Documentation (5 minutes)
bash
echo "Updating documentation..."

# Update README
echo "Deployment: $(date)" >> README.md
echo "Version: v1.2.0" >> README.md

# Update CHANGELOG
cat >> CHANGELOG.md << EOF

## [1.2.0] - $(date +%Y-%m-%d)

### Added
- Post-deployment verification checks
- Automated smoke tests
- Enhanced monitoring dashboards

### Fixed
- Reconciliation timing issues
- DLQ growth rate alerts

### Changed
- Optimized Kafka consumer groups
- Updated retry policies
EOF

# Git commit documentation updates
git add README.md CHANGELOG.md
git commit -m "Update documentation for v1.2.0 deployment"
Phase 5: Go-Live (10 minutes)
Step 5.1: Enable Production Traffic (3 minutes)
bash
echo "Enabling production traffic..."

# Route traffic to new version (Blue-Green)
kubectl patch service integration-api -n integration \
  -p '{"spec":{"selector":{"version":"green"}}}'

# Verify endpoints
kubectl get endpoints integration-api -n integration

# Test production endpoints
curl -s -o /dev/null -w "%{http_code}" https://api.integration.com/v1/health

# Check service status
kubectl get svc integration-api -n integration
Step 5.2: Verify End-to-End Flow (5 minutes)
bash
echo "Verifying end-to-end flow..."

# Trigger test extraction
curl -X POST http://scheduler:8793/api/v1/jobs/GL/trigger \
  -H "Content-Type: application/json" \
  -d '{"test": true, "batch_size": 10}'

# Wait for processing
sleep 60

# Check job status
curl -s http://scheduler:8793/api/v1/jobs/GL/status | jq

# Expected: status = "COMPLETED"

# Check data in FinSight
curl -s https://api.finsight.zetheta.com/v1/journal-entries?limit=1 \
  -H "Authorization: Bearer $ACCESS_TOKEN" | jq

# Expected: New records present
Step 5.3: Send Success Notification (2 minutes)
bash
echo "Sending success notification..."

# Send Slack notification
curl -X POST <SLACK_WEBHOOK_URL>
  -H "Content-Type: application/json" \
  -d '{
    "text": "✅ Integration Pipeline Deployment Complete",
    "blocks": [
      {
        "type": "header",
        "text": {
          "type": "plain_text",
          "text": "✅ Integration Pipeline Deployment Complete"
        }
      },
      {
        "type": "section",
        "fields": [
          {
            "type": "mrkdwn",
            "text": "*Environment:*\nProduction"
          },
          {
            "type": "mrkdwn",
            "text": "*Status:*\nSUCCESS"
          },
          {
            "type": "mrkdwn",
            "text": "*Version:*\nv1.2.0"
          },
          {
            "type": "mrkdwn",
            "text": "*Completion Time:*\n$(date)"
          }
        ]
      },
      {
        "type": "section",
        "text": {
          "type": "mrkdwn",
          "text": "*Deployment Summary:*\n- All smoke tests passed\n- Reconciliation verified\n- No critical alerts\n- End-to-end flow working"
        }
      },
      {
        "type": "section",
        "text": {
          "type": "mrkdwn",
          "text": "*Next Steps:*\n- Monitor for 24 hours\n- Review performance metrics\n- Update runbook"
        }
      }
    ]
  }'

# Update deployment tracking
echo "STATUS: completed" >> /tmp/deployment-tracking.txt
echo "END_TIME: $(date)" >> /tmp/deployment-tracking.txt
echo "✅ Deployment completed successfully!" >> /tmp/deployment-tracking.txt

# Show deployment summary
cat /tmp/deployment-tracking.txt
Deployment Summary
Deployment Details
Attribute	Value
Deployment ID	DEP-$(date +%Y%m%d-%H%M%S)
Environment	Production
Version	v1.2.0
Start Time	$(date -d "2 hours ago")
End Time	$(date)
Duration	~2 hours
Status	✅ SUCCESS
Deployed By	$(whoami)
Verification Summary
Check	Status
Smoke Tests	✅ PASS
Reconciliation	✅ PASS
Alerts	✅ No critical alerts
End-to-End Flow	✅ PASS
Quick Commands Reference
bash
# Check deployment status
kubectl get pods -n integration

# Check service health
curl -s https://api.integration.com/health

# Monitor alerts
curl -s http://prometheus:9090/api/v1/alerts

# Check reconciliation
curl -s http://reconciliation:8090/api/v1/reconciliation/status

# View logs
kubectl logs -f deployment/api-gateway -n integration
Emergency Contacts
Role	Name	Phone	Email
Integration Lead	_______________	___	___
DevOps Lead	_______________	___	___
SAP Basis Admin	_______________	___	___
Platform Engineer	_______________	___	___
VP IT Infrastructure	_______________	___	___