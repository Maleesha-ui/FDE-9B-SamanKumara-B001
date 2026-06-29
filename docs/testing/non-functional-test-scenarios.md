## Overview

This document defines 5 non-functional test scenarios for the SAP to FinSight integration pipeline, covering performance, scalability, reliability, and endurance aspects.

---

## Test Environment Specifications

| Component | Specification |
|-----------|---------------|
| **SAP System** | S/4HANA 2023 FPS02, 16 vCPUs, 64GB RAM |
| **Kafka** | 3 brokers, 8 vCPUs, 32GB RAM each |
| **Transformation Engine** | 4 nodes, 8 vCPUs, 32GB RAM each |
| **FinSight API** | 2 nodes, 8 vCPUs, 32GB RAM each |
| **Network** | 450 Mbps MPLS link (Pune to AWS Mumbai) |

---

## TST-NFR-001: Peak Load Performance

| Attribute | Value |
|-----------|-------|
| **Test ID** | TST-NFR-001 |
| **Title** | Peak Load Performance |
| **Description** | Validate system performance under peak load conditions |
| **Preconditions** | 500K GL entries available in SAP |
| **Test Type** | Performance |

### Test Configuration

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| **Records** | 500,000 GL entries | Represents peak daily volume |
| **Batch Size** | 5,000 records | Maximum batch size supported |
| **Concurrent Threads** | 10 | Simulates parallel processing |
| **Timeout** | 1 hour | Maximum allowed for extraction |
| **Data Mix** | 70% regular, 30% special periods | Realistic data distribution |

### Performance Metrics

| Metric | Target | Threshold | Critical |
|--------|--------|-----------|----------|
| Processing Time | < 2 hours | < 3 hours | > 3 hours |
| Memory Usage | < 2 GB | < 4 GB | > 4 GB |
| CPU Usage | < 70% | < 90% | > 90% |
| Error Rate | 0% | < 1% | > 1% |
| Throughput | > 500 rec/s | > 200 rec/s | < 200 rec/s |
| Network Usage | < 25% | < 40% | > 40% |

### Test Execution

```bash
# Load test command
k6 run --vus 10 --duration 30m load-test.js

# Monitor metrics
curl http://localhost:9090/metrics | grep processing_time
curl http://localhost:9090/metrics | grep memory_usage
Expected Result
text
✅ All 500K records processed within 2 hours
✅ No OOM errors
✅ No data loss
✅ Throughput > 500 records/second
✅ Network usage < 25% of 450 Mbps
TST-NFR-002: Concurrent Extraction
Attribute	Value
Test ID	TST-NFR-002
Title	Concurrent Extraction
Description	Validate system behavior when all 12 extractions run simultaneously
Preconditions	All 12 domains have data available
Test Type	Performance
Test Configuration
Parameter	Value	Rationale
Domains	All 12 domains	Complete integration scope
Concurrent Jobs	12	One per domain
Execution Mode	Parallel	Simulates production
Time Window	30 minutes	Typical extraction window
Resource Contention Points
Resource	Contention Risk	Mitigation
RFC Connections	High (max 50)	Connection pooling
Kafka Partitions	Medium	Partition by company code
Database Locks	Low	Optimistic locking
Network Bandwidth	Medium	Traffic shaping
Performance Metrics
Metric	Target	Threshold	Critical
No Deadlocks	Yes	No deadlocks	Deadlocks detected
No Resource Contention	Yes	No errors	Resource errors
All Complete	Yes	100% completion	< 100% completion
RFC Connections Used	< 45	< 50	= 50
Average Latency	< 5s	< 10s	> 10s
Test Execution
bash
# Trigger all extractions simultaneously
for domain in GL AP AR COST PROCUREMENT SALES ASSETS BANK MATERIALS BUDGET PROFIT_CENTRE EXCHANGE_RATES; do
    curl -X POST http://scheduler:8793/api/v1/jobs/$domain/trigger &
done
wait

# Check for deadlocks
grep "deadlock" /var/log/integration/*.log
Expected Result
text
✅ No deadlocks detected
✅ No resource contention
✅ All 12 extractions complete successfully
✅ RFC connections < 45
TST-NFR-003: API Latency
Attribute	Value
Test ID	TST-NFR-003
Title	API Latency
Description	Validate API latency under load
Preconditions	500 concurrent FinSight API requests
Test Type	Performance
Test Configuration
Parameter	Value	Rationale
Concurrent Requests	500	Peak concurrent load
Request Type	POST /journal-entries	Most frequent operation
Payload Size	10 KB	Average batch size
Duration	5 minutes	Sustained load
Ramp-up Time	30 seconds	Gradual increase
Latency Metrics
Percentile	Target	Threshold	Critical
P50	< 2 seconds	< 3 seconds	> 3 seconds
P90	< 4 seconds	< 6 seconds	> 6 seconds
P95	< 5 seconds	< 8 seconds	> 8 seconds
P99	< 10 seconds	< 15 seconds	> 15 seconds
P99.9	< 20 seconds	< 30 seconds	> 30 seconds
Rate Limit Metrics
Metric	Target	Threshold	Critical
HTTP 429 (Rate Limited)	0%	< 1%	> 1%
HTTP 503 (Unavailable)	0%	< 0.1%	> 0.1%
HTTP 504 (Timeout)	0%	< 0.1%	> 0.1%
Error Rate	0%	< 1%	> 1%
Test Execution
bash
# Run latency test
k6 run --vus 500 --duration 5m latency-test.js

# Sample latency-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export default function () {
    const payload = JSON.stringify({
        entries: generateBatch(10)
    });
    
    const res = http.post('https://api.finsight.zetheta.com/v1/journal-entries', payload, {
        headers: {
            'Authorization': `Bearer ${__ENV.ACCESS_TOKEN}`,
            'Content-Type': 'application/json'
        }
    });
    
    check(res, {
        'status is 201': (r) => r.status === 201,
        'response time < 5s': (r) => r.timings.duration < 5000
    });
}
Expected Result
text
✅ P95 latency < 5 seconds
✅ No HTTP 429 beyond expected rate limits
✅ All requests complete successfully
✅ Error rate < 1%
TST-NFR-004: Scalability Test
Attribute	Value
Test ID	TST-NFR-004
Title	Scalability Test
Description	Validate system scalability with increasing data volume
Preconditions	Incremental data from 1K to 100K records
Test Type	Scalability
Test Configuration
Parameter	Value	Rationale
Record Increments	1K, 5K, 10K, 25K, 50K, 75K, 100K	Gradual increase
Measurement	Processing time per record	Linear scaling check
Repeat Count	3 times per increment	Reduce variance
Warm-up	1 run per increment	Cache initialization
Scalability Metrics
Metric	Target	Threshold	Critical
Linear Throughput	Yes, R² > 0.95	R² > 0.90	R² < 0.90
No Degradation	Below threshold	< 20% increase	> 20% increase
Scaling Ceiling	> 200K records	> 100K records	< 100K records
Memory Scaling	< O(n)	< O(n log n)	> O(n log n)
Test Results Template
Records	Run 1 (s)	Run 2 (s)	Run 3 (s)	Avg (s)	Rec/s	Mem (MB)	CPU (%)
1,000	2.1	2.0	2.2	2.1	476	128	15
5,000	9.8	9.5	10.1	9.8	510	256	25
10,000	18.5	18.2	19.0	18.6	538	384	35
25,000	45.2	44.8	46.0	45.3	552	512	45
50,000	90.5	89.2	91.8	90.5	552	768	55
75,000	138.0	136.5	140.2	138.2	543	1024	65
100,000	188.5	186.8	190.2	188.5	531	1280	72
Expected Result
text
✅ Linear throughput scaling (R² > 0.95)
✅ No degradation below ceiling
✅ Scaling ceiling > 100K records
✅ Memory scaling < O(n log n)
TST-NFR-005: 24-Hour Endurance
Attribute	Value
Test ID	TST-NFR-005
Title	24-Hour Endurance
Description	Validate system stability over 24 hours of continuous operation
Preconditions	Continuous data flow for 24 hours
Test Type	Reliability
Test Configuration
Parameter	Value	Rationale
Duration	24 hours	One full day of production
Data Rate	Steady stream	Simulates normal traffic
Monitoring	All metrics tracked	Complete visibility
Checkpoints	Every 6 hours	Progress verification
Maintenance	None during test	Production conditions
Endurance Metrics
Metric	Target	Threshold	Critical
No Memory Leaks	Yes	No increasing memory	Memory leak detected
No Connection Exhaustion	Yes	No connection errors	Connection errors
Stable Throughput	Yes	< 20% variance	> 20% variance
No Service Restarts	Yes	0 restarts	Any restart
Log Growth Rate	< 1 GB/day	< 2 GB/day	> 2 GB/day
Error Rate Trend	Flat	< 10% increase	> 10% increase
Monitoring Dashboard
text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    24-Hour Endurance Test Monitoring                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Memory Usage Over Time                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ 500 MB ┤                                                           │    │
│  │ 400 MB ┤    ╭────────────────────────────────────────╮             │    │
│  │ 300 MB ┤   ╭╯                                        ╰╮            │    │
│  │ 200 MB ┤  ╭╯                                          ╰╮           │    │
│  │ 100 MB ┤─╭╯                                            ╰────────────│    │
│  │   0 MB └────────────────────────────────────────────────────────────│    │
│  │        00:00  04:00  08:00  12:00  16:00  20:00  24:00            │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  Throughput Over Time                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ 600 rec/s ┤                                                       │    │
│  │ 500 rec/s ┤    ██████████████████████████████████████████████████ │    │
│  │ 400 rec/s ┤    ██████████████████████████████████████████████████ │    │
│  │ 300 rec/s ┤    ██████████████████████████████████████████████████ │    │
│  │ 200 rec/s ┤    ██████████████████████████████████████████████████ │    │
│  │ 100 rec/s ┤    ██████████████████████████████████████████████████ │    │
│  │   0 rec/s └────────────────────────────────────────────────────────│    │
│  │        00:00  04:00  08:00  12:00  16:00  20:00  24:00            │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  Error Rate Over Time                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ 2.0% ┤                                                           │    │
│  │ 1.5% ┤                                                           │    │
│  │ 1.0% ┤           ╭─╮                                             │    │
│  │ 0.5% ┤          ╭╯ ╰╮                                           │    │
│  │ 0.0% ┤─────────╭╯   ╰────────────────────────────────────────────│    │
│  │        00:00  04:00  08:00  12:00  16:00  20:00  24:00            │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
Checkpoint Validation
Checkpoint	Time	Items to Verify	Status
Checkpoint 1	06:00	Memory, Throughput, Errors	___
Checkpoint 2	12:00	Memory, Throughput, Errors	___
Checkpoint 3	18:00	Memory, Throughput, Errors	___
Checkpoint 4	24:00	All metrics, Final report	___
Expected Result
text
✅ No memory leaks detected
✅ No connection exhaustion
✅ Stable throughput throughout
✅ Zero service restarts
✅ Log growth < 1 GB/day
✅ Error rate trend flat
Test Execution Summary
Test Execution Matrix
Test ID	Title	Type	Status	Duration	Result
TST-NFR-001	Peak Load Performance	Performance	___	___	___
TST-NFR-002	Concurrent Extraction	Performance	___	___	___
TST-NFR-003	API Latency	Performance	___	___	___
TST-NFR-004	Scalability Test	Scalability	___	___	___
TST-NFR-005	24-Hour Endurance	Reliability	___	___	___
Pass/Fail Criteria
Category	Pass Criteria	Fail Criteria
Performance	All metrics within target	Any metric exceeds critical
Scalability	R² > 0.95, linear scaling	R² < 0.90, non-linear
Reliability	Zero restarts, no leaks	Any restart or leak
Stability	< 20% variance	> 20% variance
Test Environment Cleanup
Post-Test Actions
Action	Command	Owner
Stop Load Test	k6 stop	QA Engineer
Collect Logs	kubectl logs > test-logs/	QA Engineer
Collect Metrics	curl http://prometheus:9090/api/v1/query	QA Engineer
Reset Database	psql -c "TRUNCATE TABLE test_data"	DBA
Clean Kafka Topics	kafka-topics --delete --topic test-*	