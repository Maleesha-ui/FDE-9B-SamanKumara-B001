# Monitoring Dashboard Panels Specification

## Overview

This document defines the 12 monitoring dashboard panels for the SAP to FinSight integration pipeline. Each panel includes metrics, data sources, refresh intervals, and alert thresholds.

---

## Panel MON-001: Pipeline Health Overview

| Attribute | Value |
|-----------|-------|
| **Panel ID** | MON-001 |
| **Title** | Pipeline Health Overview |
| **Type** | Status Aggregation |
| **Data Source** | Prometheus - Integration Status Metrics |

### Metrics
```promql
# Status aggregation
pipeline_status{environment="production"}

# Status values:
# 0 = GREEN (All domains synced within SLA)
# 1 = YELLOW (Any domain >2hrs stale)
# 2 = RED (Any domain >4hrs stale or last failed)