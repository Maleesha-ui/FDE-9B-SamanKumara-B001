# Circuit Breaker State Machine

## Overview

The Circuit Breaker pattern prevents cascade failures in distributed systems by stopping calls to a failing service after a threshold of consecutive failures. This document defines the circuit breaker implementation for the SAP to FinSight integration pipeline.

---

## States

### 1. CLOSED (Normal Operation)
- All requests pass through to the downstream service
- Failures are counted against a threshold
- When threshold is exceeded → transitions to OPEN

### 2. OPEN (Circuit Open)
- All requests fail immediately without calling the service
- A fallback response is returned (if configured)
- After timeout period → transitions to HALF-OPEN
- Prevents cascading failures and resource exhaustion

### 3. HALF-OPEN (Testing)
- Limited requests allowed through to test if service recovered
- If successful → transitions to CLOSED
- If failure occurs → transitions back to OPEN
- Prevents overwhelming a recovering service

---

## State Machine Diagram

---

## Configuration Parameters

| Parameter | Default Value | Description |
|-----------|---------------|-------------|
| `failureThreshold` | 5 | Consecutive failures before circuit opens |
| `successThreshold` | 3 | Successes needed in HALF-OPEN to close |
| `timeoutOpen` | 30 seconds | Time in OPEN state before transitioning to HALF-OPEN |
| `halfOpenMaxRequests` | 1 | Maximum requests allowed in HALF-OPEN state |
| `maxConcurrentCalls` | 10 | Maximum concurrent calls allowed |
| `ignoreExceptions` | [] | Exceptions that don't count as failures |

---

## Service-Specific Configurations

| Service | Failure Threshold | Timeout | Half-Open Requests |
|---------|-------------------|---------|-------------------|
| SAP RFC | 3 | 30s | 1 |
| FinSight API | 5 | 60s | 2 |
| Kafka | 3 | 20s | 1 |
| Snowflake | 5 | 45s | 2 |

---

## Implementation Code

### Python Implementation

```python
import time
import threading
import logging
from enum import Enum
from typing import Optional, Callable, Any, List
from dataclasses import dataclass

class CircuitState(Enum):
    """Circuit breaker states."""
    CLOSED = "closed"
    OPEN = "open"
    HALF_OPEN = "half_open"

@dataclass
class CircuitBreakerConfig:
    """Configuration for circuit breaker."""
    failure_threshold: int = 5
    success_threshold: int = 3
    timeout_seconds: int = 30
    half_open_max_requests: int = 1
    max_concurrent_calls: int = 10
    ignore_exceptions: List[type] = None

class CircuitBreaker:
    """
    Circuit breaker implementation with state management.
    """
    
    def __init__(self, name: str, config: CircuitBreakerConfig = None):
        self.name = name
        self.config = config or CircuitBreakerConfig()
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time = None
        self.last_state_change_time = None
        self.half_open_requests = 0
        self._lock = threading.Lock()
        self.metrics = {
            'total_calls': 0,
            'successful_calls': 0,
            'failed_calls': 0,
            'rejected_calls': 0
        }
        
        # Default ignore exceptions
        if self.config.ignore_exceptions is None:
            self.config.ignore_exceptions = []
    
    def _is_exception_ignored(self, exception: Exception) -> bool:
        """Check if exception should be ignored."""
        for ignored in self.config.ignore_exceptions:
            if isinstance(exception, ignored):
                return True
        return False
    
    def _should_open(self) -> bool:
        """Check if circuit should transition to OPEN."""
        return self.failure_count >= self.config.failure_threshold
    
    def _should_close(self) -> bool:
        """Check if circuit should transition to CLOSED."""
        return self.success_count >= self.config.success_threshold
    
    def _should_allow_request(self) -> bool:
        """Check if request should be allowed through."""
        with self._lock:
            if self.state == CircuitState.CLOSED:
                return True
            
            if self.state == CircuitState.OPEN:
                # Check if timeout has expired
                if self.last_state_change_time is None:
                    return False
                
                elapsed = time.time() - self.last_state_change_time
                if elapsed >= self.config.timeout_seconds:
                    self.state = CircuitState.HALF_OPEN
                    self.half_open_requests = 0
                    self.last_state_change_time = time.time()
                    logging.info(f"Circuit breaker '{self.name}' transitioning to HALF_OPEN")
                    return True
                return False
            
            if self.state == CircuitState.HALF_OPEN:
                if self.half_open_requests < self.config.half_open_max_requests:
                    self.half_open_requests += 1
                    return True
                return False
            
            return False
    
    def _record_success(self):
        """Record successful execution."""
        with self._lock:
            self.metrics['successful_calls'] += 1
            
            if self.state == CircuitState.HALF_OPEN:
                self.success_count += 1
                if self._should_close():
                    self.state = CircuitState.CLOSED
                    self.failure_count = 0
                    self.success_count = 0
                    self.last_state_change_time = time.time()
                    logging.info(f"Circuit breaker '{self.name}' transitioning to CLOSED")
            elif self.state == CircuitState.CLOSED:
                self.failure_count = 0  # Reset failures on success
    
    def _record_failure(self, error: Exception):
        """Record failed execution."""
        with self._lock:
            self.metrics['failed_calls'] += 1
            self.last_failure_time = time.time()
            
            if self.state == CircuitState.HALF_OPEN:
                self.state = CircuitState.OPEN
                self.success_count = 0
                self.half_open_requests = 0
                self.last_state_change_time = time.time()
                logging.warning(f"Circuit breaker '{self.name}' transitioning to OPEN (failure in HALF-OPEN)")
            elif self.state == CircuitState.CLOSED:
                self.failure_count += 1
                if self._should_open():
                    self.state = CircuitState.OPEN
                    self.last_state_change_time = time.time()
                    logging.warning(f"Circuit breaker '{self.name}' transitioning to OPEN (threshold reached)")
    
    def execute(self, operation: Callable, fallback: Optional[Callable] = None, *args, **kwargs) -> Any:
        """
        Execute operation with circuit breaker protection.
        
        Args:
            operation: Function to execute
            fallback: Fallback function if circuit is OPEN
            *args, **kwargs: Arguments to pass to operation
        
        Returns:
            Any: Result of operation or fallback
        
        Raises:
            RuntimeError: If circuit is OPEN and no fallback
            Exception: Any exception from operation
        """
        self.metrics['total_calls'] += 1
        
        # Check if request should be allowed
        if not self._should_allow_request():
            self.metrics['rejected_calls'] += 1
            logging.warning(f"Circuit breaker '{self.name}' is OPEN - rejecting request")
            
            if fallback:
                return fallback()
            raise RuntimeError(f"Circuit breaker '{self.name}' is OPEN")
        
        try:
            result = operation(*args, **kwargs)
            self._record_success()
            return result
        except Exception as e:
            # Check if exception should be ignored
            if self._is_exception_ignored(e):
                logging.warning(f"Circuit breaker '{self.name}' ignoring exception: {e}")
                return operation(*args, **kwargs)
            
            self._record_failure(e)
            raise
    
    def get_state(self) -> dict:
        """Get current circuit state and metrics."""
        return {
            'name': self.name,
            'state': self.state.value,
            'failure_count': self.failure_count,
            'success_count': self.success_count,
            'last_failure_time': self.last_failure_time,
            'last_state_change_time': self.last_state_change_time,
            'metrics': self.metrics
        }
    
    def reset(self):
        """Reset circuit breaker to CLOSED state."""
        with self._lock:
            self.state = CircuitState.CLOSED
            self.failure_count = 0
            self.success_count = 0
            self.half_open_requests = 0
            self.last_state_change_time = time.time()
            logging.info(f"Circuit breaker '{self.name}' manually reset to CLOSED")


# Circuit Breaker Registry
class CircuitBreakerRegistry:
    """Registry for managing circuit breakers by service."""
    
    _instance = None
    _breakers = {}
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
    
    @classmethod
    def get(cls, service_name: str) -> CircuitBreaker:
        """Get or create circuit breaker for service."""
        if service_name not in cls._breakers:
            config = cls._get_config_for_service(service_name)
            cls._breakers[service_name] = CircuitBreaker(service_name, config)
        return cls._breakers[service_name]
    
    @classmethod
    def _get_config_for_service(cls, service_name: str) -> CircuitBreakerConfig:
        """Get configuration for specific service."""
        configs = {
            'sap-rfc': CircuitBreakerConfig(
                failure_threshold=3,
                timeout_seconds=30,
                half_open_max_requests=1,
                max_concurrent_calls=50
            ),
            'finsight-api': CircuitBreakerConfig(
                failure_threshold=5,
                timeout_seconds=60,
                half_open_max_requests=2,
                max_concurrent_calls=100
            ),
            'kafka': CircuitBreakerConfig(
                failure_threshold=3,
                timeout_seconds=20,
                half_open_max_requests=1,
                max_concurrent_calls=50
            ),
            'snowflake': CircuitBreakerConfig(
                failure_threshold=5,
                timeout_seconds=45,
                half_open_max_requests=2,
                max_concurrent_calls=25
            )
        }
        return configs.get(service_name, CircuitBreakerConfig())
    
    @classmethod
    def get_all_states(cls) -> dict:
        """Get states of all circuit breakers."""
        return {
            name: breaker.get_state()
            for name, breaker in cls._breakers.items()
        }


# Usage Example
def usage_example():
    """Example of using circuit breaker."""
    import requests
    
    # Get circuit breaker for FinSight API
    cb = CircuitBreakerRegistry.get('finsight-api')
    
    def call_finsight_api():
        response = requests.post(
            'https://api.finsight.zetheta.com/v1/journal-entries',
            json={'entries': [...]},
            timeout=30
        )
        response.raise_for_status()
        return response.json()
    
    def fallback_response():
        return {'status': 'service_unavailable', 'error': 'Circuit breaker open'}
    
    try:
        result = cb.execute(call_finsight_api, fallback_response)
        print(f"Result: {result}")
    except Exception as e:
        print(f"Error: {e}")
    
    # Get current state
    state = cb.get_state()
    print(f"Circuit State: {state}")