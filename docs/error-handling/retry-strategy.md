# Retry Strategy Specification

## Overview

This document defines the retry strategy for the integration pipeline, including exponential backoff with jitter, retry limits, and circuit breaker configuration.

---

## 1. Exponential Backoff with Jitter

### Formula

### Parameters

| Parameter | Value | Description |
|-----------|-------|-------------|
| Base | 2 seconds | Initial delay |
| Cap | 60 seconds | Maximum delay |
| Jitter | random(0.8, 1.2) | Randomization factor |
| Max Attempts | 3 | Maximum retry attempts |

### Retry Schedule

| Attempt | Delay | With Jitter Range |
|---------|-------|-------------------|
| 1 | 2s | 1.6s - 2.4s |
| 2 | 4s | 3.2s - 4.8s |
| 3 | 8s | 6.4s - 9.6s |
| 4 | 16s | 12.8s - 19.2s |
| 5 | 32s | 25.6s - 38.4s |
| 6 | 60s (capped) | 48s - 72s |

### Implementation

```python
import random
import time
from typing import Optional

def exponential_backoff_with_jitter(
    attempt: int,
    base_delay: float = 2.0,
    cap: float = 60.0,
    jitter_factor: float = 0.2
) -> float:
    """
    Calculate wait time with exponential backoff and jitter.
    
    Args:
        attempt: Current retry attempt (0-indexed)
        base_delay: Base delay in seconds
        cap: Maximum delay in seconds
        jitter_factor: Randomization factor (0.0 - 0.5)
    
    Returns:
        float: Wait time in seconds
    """
    # Calculate exponential delay
    delay = base_delay * (2 ** attempt)
    
    # Apply cap
    delay = min(delay, cap)
    
    # Apply jitter
    jitter_range = delay * jitter_factor
    jitter = random.uniform(-jitter_range, jitter_range)
    wait_time = delay + jitter
    
    return max(0, wait_time)

# Usage
def retry_operation(operation, max_attempts: int = 3):
    for attempt in range(max_attempts):
        try:
            return operation()
        except TransientError as e:
            if attempt == max_attempts - 1:
                raise
            wait_time = exponential_backoff_with_jitter(attempt)
            time.sleep(wait_time)