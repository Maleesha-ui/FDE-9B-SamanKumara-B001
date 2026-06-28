function updateStats(stats) {
    document.getElementById('pending-count').textContent = stats.pending;
    document.getElementById('reviewed-count').textContent = stats.reviewed;
    document.getElementById('reprocessed-count').textContent = stats.reprocessed;
    document.getElementById('resolved-count').textContent = stats.resolved;
    document.getElementById('total-count').textContent = stats.total;
    document.getElementById('failed-count').textContent = stats.failed;
}

function viewRecord(recordId) {
    // Open record details modal
    fetch(`/api/dlq/records/${recordId}`)
        .then(response => response.json())
        .then(record => {
            showRecordModal(record);
        });
}

function reprocess(recordId) {
    if (confirm('Are you sure you want to reprocess this record?')) {
        fetch(`/api/dlq/records/${recordId}/reprocess`, {
            method: 'POST'
        })
        .then(response => response.json())
        .then(result => {
            if (result.success) {
                alert('Record reprocessed successfully');
                loadRecords();
            } else {
                alert('Failed to reprocess: ' + result.error);
            }
        });
    }
}

function deleteRecord(recordId) {
    if (confirm('Are you sure you want to delete this record?')) {
        fetch(`/api/dlq/records/${recordId}`, {
            method: 'DELETE'
        })
        .then(response => response.json())
        .then(result => {
            if (result.success) {
                alert('Record deleted');
                loadRecords();
            }
        });
    }
}

function applyFilters() {
    const domain = document.getElementById('domain-filter').value;
    const error = document.getElementById('error-filter').value;
    const status = document.getElementById('status-filter').value;
    const from = document.getElementById('date-from').value;
    const to = document.getElementById('date-to').value;
    
    fetch(`/api/dlq/records?domain=${domain}&error=${error}&status=${status}&from=${from}&to=${to}`)
        .then(response => response.json())
        .then(data => {
            renderTable(data.records);
        });
}

function refreshQueue() {
    loadRecords();
    document.getElementById('last-refresh').textContent = new Date().toLocaleString();
}

// Load on page load
document.addEventListener('DOMContentLoaded', loadRecords);
</script>
</body>
</html>


def reprocess_dlq_record(record_id: str) -> Dict[str, Any]:
    """
    Reprocess a DLQ record after manual review.
    
    Args:
        record_id: DLQ record ID
    
    Returns:
        Dict: Reprocessing result
    """
    # Get record from database
    record = db.get_dlq_record(record_id)
    
    if not record:
        return {'success': False, 'error': 'Record not found'}
    
    if record['status'] == 'REPROCESSED':
        return {'success': False, 'error': 'Record already reprocessed'}
    
    try:
        # Extract original record data
        original_data = record['record_data']
        domain = record['domain']
        
        # Log reprocessing attempt
        log_reprocessing_attempt(record_id)
        
        # Apply validation
        validation_result = validate_record(original_data, domain)
        
        if not validation_result['valid']:
            db.update_status(record_id, 'FAILED', validation_result['errors'])
            return {'success': False, 'error': 'Validation failed', 'details': validation_result['errors']}
        
        # Apply transformations
        transformed = transform_record(original_data, domain)
        
        # Load to FinSight
        load_result = load_to_finsight(transformed, domain)
        
        if load_result['success']:
            db.update_status(record_id, 'REPROCESSED')
            create_audit_log(record_id, 'REPROCESSED', 'Manual reprocess successful')
            return {'success': True, 'message': 'Record reprocessed successfully'}
        else:
            db.update_status(record_id, 'FAILED', load_result['error'])
            return {'success': False, 'error': load_result['error']}
            
    except Exception as e:
        db.update_status(record_id, 'FAILED', str(e))
        return {'success': False, 'error': str(e)}

def batch_reprocess(record_ids: List[str]) -> Dict[str, Any]:
    """
    Batch reprocess multiple DLQ records.
    
    Args:
        record_ids: List of record IDs
    
    Returns:
        Dict: Batch reprocessing results
    """
    results = {
        'total': len(record_ids),
        'success': 0,
        'failed': 0,
        'errors': []
    }
    
    for record_id in record_ids:
        try:
            result = reprocess_dlq_record(record_id)
            if result['success']:
                results['success'] += 1
            else:
                results['failed'] += 1
                results['errors'].append({
                    'record_id': record_id,
                    'error': result.get('error', 'Unknown error')
                })
        except Exception as e:
            results['failed'] += 1
            results['errors'].append({
                'record_id': record_id,
                'error': str(e)
            })
    
    return results


    from prometheus_client import Gauge, Counter

# Metrics
dlq_total_records = Gauge(
    'dlq_total_records',
    'Total records in Dead Letter Queue',
    ['domain']
)

dlq_pending_records = Gauge(
    'dlq_pending_records',
    'Pending records in Dead Letter Queue',
    ['domain']
)

dlq_reprocessed_records = Counter(
    'dlq_reprocessed_records_total',
    'Total records reprocessed from DLQ',
    ['domain', 'status']
)

dlq_record_age = Gauge(
    'dlq_record_age_days',
    'Age of DLQ records in days',
    ['domain', 'status']
)

dlq_error_count = Gauge(
    'dlq_error_count',
    'Count of DLQ records by error type',
    ['domain', 'error_code']
)

┌─────────────────────────────────────────────────────────────────────────┐
│ Dead Letter Queue Monitoring Dashboard                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ╔═══════════╗ ╔═══════════╗ ╔═══════════╗ ╔═══════════╗              │
│  ║ Total DLQ ║ ║ Pending   ║ ║ Reprocess ║ ║ Avg Age   ║              │
│  ║  1,247    ║ ║  1,032    ║ ║    58     ║ ║  2.3 days ║              │
│  ╚═══════════╝ ╚═══════════╝ ╚═══════════╝ ╚═══════════╝              │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ DLQ by Domain                                                   │    │
│  │ GL:  ████████████████████ 847                                   │    │
│  │ AP:  ████████████ 312                                           │    │
│  │ AR:  █████ 88                                                   │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ DLQ by Error Code                                               │    │
│  │ ERR-MAP-003: ████████████████████████ 456                       │    │
│  │ ERR-MAP-001: ██████████████ 312                                 │    │
│  │ ERR-MAP-004: █████████ 189                                      │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ DLQ Trend (Last 7 Days)                                         │    │
│  │     200 │                                                    █   │    │
│  │     150 │                                  ██  ██████  ████  │    │
│  │     100 │                      ██████  ████████████████████  │    │
│  │      50 │  ██████  ████████████████████████████████████████  │    │
│  │       0 └────────────────────────────────────────────────────  │    │
│  │        Mon Tue Wed Thu Fri Sat Sun                            │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

alerts:
  - name: dlq_pending_high
    condition: dlq_pending_records{domain!=""} > 500
    for: 5m
    severity: critical
    annotations:
      summary: "High DLQ pending count - {{ $labels.domain }}"
      description: "{{ $labels.domain }} has {{ $value }} pending DLQ records"
      action: "Review DLQ dashboard and reprocess records"

  - name: dlq_total_high
    condition: dlq_total_records > 1000
    for: 10m
    severity: warning
    annotations:
      summary: "High total DLQ count"
      description: "Total DLQ records: {{ $value }}"
      action: "Investigate root cause and clean up DLQ"

  - name: dlq_old_records
    condition: dlq_record_age_days > 7
    for: 1h
    severity: warning
    annotations:
      summary: "Old DLQ records detected"
      description: "Records older than 7 days: {{ $value }}"
      action: "Review and resolve old records"