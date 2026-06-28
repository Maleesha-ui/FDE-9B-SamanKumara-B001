# Error Notification Matrix

## Overview

This document defines the notification strategy for different error types in the SAP to FinSight integration pipeline, including channels, recipients, and SLAs.

---

## Notification Channels

| Channel | Purpose | Response Time | Escalation |
|---------|---------|---------------|------------|
| **PagerDuty** | Critical alerts (P1, P2) | < 5 minutes | Auto-escalate every 15 min |
| **Email** | Standard notifications (P3, P4) | < 1 hour | Manual escalation only |
| **Slack** | Team notifications, status updates | < 30 minutes | @mention on-call engineer |
| **Dashboard** | Monitoring visualizations | Real-time | Auto-refresh every 30s |
| **SMS** | Backup for P1 alerts | < 5 minutes | If PagerDuty unacknowledged |
| **Phone Call** | Emergency P1 alerts | < 2 minutes | If SMS unacknowledged |

---

## Error Priority Definitions

### P1 - Critical (System Down / Data Loss Risk)

**Response SLA:** 15 minutes  
**Resolution SLA:** 1 hour  
**Escalation:** Every 15 minutes

| Error Code | Description | Trigger Conditions |
|------------|-------------|-------------------|
| ERR-EXT-003 | ODP subscription invalidated | Provider structure changed, no data extraction |
| ERR-LOAD-005 | Authentication failure (all retries) | OAuth token refresh failed after 3 attempts |
| ERR-SYS-001 | Kafka broker unreachable | All Kafka brokers down, no event streaming |
| ERR-SYS-002 | Disk space < 10% | Storage capacity critical |
| ERR-SYS-003 | Memory exhaustion | Container OOM, service crash |
| ERR-RECON-002 | Checksum variance > 1% | Significant data inconsistency detected |

### P2 - High (Major Functionality Affected)

**Response SLA:** 1 hour  
**Resolution SLA:** 4 hours  
**Escalation:** Every 30 minutes

| Error Code | Description | Trigger Conditions |
|------------|-------------|-------------------|
| ERR-EXT-001 | RFC timeout after retries | 3 consecutive timeouts, no SAP data |
| ERR-EXT-002 | RFC pool exhausted | All 50 connections in use |
| ERR-EXT-004 | ODP empty response | No data returned, possible issue |
| ERR-LOAD-001 | FinSight 429 rate limit | Rate limit exceeded, data delay |
| ERR-LOAD-002 | FinSight 503 unavailable | Service maintenance or overload |
| ERR-LOAD-004 | Duplicate entry (idempotent) | Duplicate records detected |
| ERR-MAP-003 | Cost centre not found (bulk) | >10% of records have missing CC |
| ERR-RECON-001 | Record count mismatch | Source vs target count mismatch |
| ERR-RECON-003 | Referential integrity violation | Missing master data references |

### P3 - Medium (Minor Functionality Affected)

**Response SLA:** 4 hours  
**Resolution SLA:** 24 hours  
**Escalation:** Every 2 hours

| Error Code | Description | Trigger Conditions |
|------------|-------------|-------------------|
| ERR-MAP-001 | NULL value with default | NULL field, default applied |
| ERR-MAP-002 | Invalid date format | Date format corrected automatically |
| ERR-MAP-004 | Invalid currency code | Currency code not in ISO 4217 |
| ERR-MAP-005 | Stale exchange rate | Rate > 30 days old |
| ERR-MAP-006 | Transformation warning | Non-critical transformation issue |
| ERR-LOAD-006 | Partial batch failure | Some records failed, some succeeded |

### P4 - Low (Cosmetic / Warning)

**Response SLA:** 24 hours  
**Resolution SLA:** 48 hours  
**Escalation:** No auto-escalation

| Error Code | Description | Trigger Conditions |
|------------|-------------|-------------------|
| Validation warnings | Minor data quality issues | Non-critical validation failures |
| Logging warnings | Non-critical logging issues | Log file warnings |
| Monitoring warnings | Performance degradation | Minor performance issues |

---

## Notification Templates

### 1. PagerDuty Alert Template

```json
{
    "event_action": "trigger",
    "payload": {
        "summary": "[P1] {service_name} - {error_code}",
        "severity": "critical",
        "source": "{source_system}",
        "component": "{component_name}",
        "group": "{group_name}",
        "class": "{class_name}",
        "custom_details": {
            "error_code": "{error_code}",
            "error_message": "{error_message}",
            "batch_id": "{batch_id}",
            "domain": "{domain}",
            "record_id": "{record_id}",
            "timestamp": "{timestamp}",
            "environment": "{environment}",
            "error_details": "{error_details}",
            "impact": "{impact_description}",
            "recommended_action": "{recommended_action}",
            "dashboard_url": "{dashboard_url}",
            "runbook_url": "{runbook_url}"
        }
    },
    "links": [
        {
            "href": "{dashboard_url}",
            "text": "View in Dashboard"
        },
        {
            "href": "{runbook_url}",
            "text": "View Runbook"
        }
    ],
    "images": [
        {
            "src": "{image_url}",
            "alt": "Error Graph"
        }
    ]
}
<!DOCTYPE html>
<html>
<head>
    <style>
        .email-container {
            font-family: Arial, sans-serif;
            max-width: 800px;
            margin: 0 auto;
            padding: 20px;
            border: 1px solid #ddd;
        }
        .header {
            background: #2c3e50;
            color: white;
            padding: 20px;
            border-radius: 8px 8px 0 0;
        }
        .header h1 { margin: 0; }
        .header .priority { font-size: 24px; font-weight: bold; }
        .priority-P1 { color: #dc3545; }
        .priority-P2 { color: #fd7e14; }
        .priority-P3 { color: #ffc107; }
        .priority-P4 { color: #6c757d; }
        .body { padding: 20px; background: #f8f9fa; }
        .section { margin-bottom: 20px; }
        .section-title { font-weight: bold; color: #2c3e50; margin-bottom: 10px; }
        .detail-row { padding: 8px 0; border-bottom: 1px solid #eee; }
        .detail-label { font-weight: bold; width: 150px; display: inline-block; }
        .button {
            display: inline-block;
            padding: 10px 20px;
            margin: 5px;
            border-radius: 4px;
            text-decoration: none;
            color: white;
        }
        .button-primary { background: #4d96ff; }
        .button-success { background: #28a745; }
        .button-danger { background: #dc3545; }
        .footer {
            padding: 20px;
            font-size: 12px;
            color: #666;
            text-align: center;
            border-top: 1px solid #ddd;
        }
    </style>
</head>
<body>
    <div class="email-container">
        <!-- Header -->
        <div class="header">
            <h1>🔔 Integration Alert - {error_code}</h1>
            <div class="priority priority-{priority}">Priority: {priority}</div>
            <p>Service: {service_name} | Domain: {domain}</p>
        </div>
        
        <!-- Body -->
        <div class="body">
            <!-- Error Summary -->
            <div class="section">
                <div class="section-title">Error Summary</div>
                <div class="detail-row">
                    <span class="detail-label">Error Code:</span>
                    {error_code}
                </div>
                <div class="detail-row">
                    <span class="detail-label">Message:</span>
                    {error_message}
                </div>
                <div class="detail-row">
                    <span class="detail-label">Timestamp:</span>
                    {timestamp}
                </div>
                <div class="detail-row">
                    <span class="detail-label">Environment:</span>
                    {environment}
                </div>
            </div>
            
            <!-- Impact -->
            <div class="section">
                <div class="section-title">Impact</div>
                <div class="detail-row">{impact_description}</div>
                <div class="detail-row">
                    <span class="detail-label">Affected Records:</span>
                    {affected_count}
                </div>
                <div class="detail-row">
                    <span class="detail-label">Error Rate:</span>
                    {error_rate}%
                </div>
            </div>
            
            <!-- Batch Details -->
            <div class="section">
                <div class="section-title">Batch Details</div>
                <div class="detail-row">
                    <span class="detail-label">Batch ID:</span>
                    {batch_id}
                </div>
                <div class="detail-row">
                    <span class="detail-label">Domain:</span>
                    {domain}
                </div>
                <div class="detail-row">
                    <span class="detail-label">Record ID:</span>
                    {record_id}
                </div>
            </div>
            
            <!-- Recommended Action -->
            <div class="section">
                <div class="section-title">Recommended Actions</div>
                <ul>
                    <li>{action_1}</li>
                    <li>{action_2}</li>
                    <li>{action_3}</li>
                </ul>
            </div>
            
            <!-- Action Buttons -->
            <div class="section">
                <div class="section-title">Quick Actions</div>
                <a href="{dashboard_url}" class="button button-primary">📊 View Dashboard</a>
                <a href="{dlq_url}" class="button button-warning">📋 Review DLQ</a>
                <a href="{runbook_url}" class="button button-success">📖 View Runbook</a>
            </div>
        </div>
        
        <!-- Footer -->
        <div class="footer">
            <p>This is an automated notification from the Integration Monitoring System.</p>
            <p>Contact: integration-support@zetheta.com | SLA: {response_sla}</p>
            <p>© 2026 Zetheta Algorithms Private Limited</p>
        </div>
    </div>
</body>
</html>
{
    "blocks": [
        {
            "type": "header",
            "text": {
                "type": "plain_text",
                "text": "🔔 Integration Alert - {priority}: {error_code}",
                "emoji": true
            }
        },
        {
            "type": "section",
            "fields": [
                {
                    "type": "mrkdwn",
                    "text": "*Service:*\n{service_name}"
                },
                {
                    "type": "mrkdwn",
                    "text": "*Domain:*\n{domain}"
                },
                {
                    "type": "mrkdwn",
                    "text": "*Priority:*\n{priority} - {sla}"
                },
                {
                    "type": "mrkdwn",
                    "text": "*Timestamp:*\n{timestamp}"
                }
            ]
        },
        {
            "type": "section",
            "text": {
                "type": "mrkdwn",
                "text": "*Error:*\n{error_message}"
            }
        },
        {
            "type": "section",
            "text": {
                "type": "mrkdwn",
                "text": "*Impact:*\n{impact_description}"
            }
        },
        {
            "type": "actions",
            "elements": [
                {
                    "type": "button",
                    "text": {
                        "type": "plain_text",
                        "text": "📊 View Dashboard"
                    },
                    "url": "{dashboard_url}"
                },
                {
                    "type": "button",
                    "text": {
                        "type": "plain_text",
                        "text": "📋 Review DLQ"
                    },
                    "url": "{dlq_url}",
                    "style": "danger"
                },
                {
                    "type": "button",
                    "text": {
                        "type": "plain_text",
                        "text": "✅ Acknowledge"
                    },
                    "value": "acknowledge"
                }
            ]
        }
    ]
}
