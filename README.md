
# Spring-Resilient-Circuit-Breaker-Api

## Overview

This project is a **multi-microservice demonstration** of **resilient external API calls** using **Resilience4j** circuit breakers in **Spring Boot 3.x**. It showcases how to protect services from cascading failures when integrating with unreliable or slow external systems by implementing **circuit breaker**, **timeout**, **retry**, **rate limiter**, and **fallback** patterns.

The focus is on **production-grade resilience**: configurable policies, shared state, metrics, and observability — preventing one failing dependency from bringing down the entire system.

## Real-World Scenario (Simulated)

In applications integrating external APIs like **weather services** (OpenWeatherMap, WeatherAPI):
- External services can be slow, rate-limited, or temporarily unavailable.
- A single slow call can exhaust thread pools, causing widespread latency or outages (cascading failure).
- Business continuity requires graceful degradation: fallbacks, caching, or default responses.

We simulate a weather dashboard system where a gateway calls unreliable downstream/external services, protected by Resilience4j to maintain availability even when dependencies fail.

## Microservices Involved

| Service                  | Responsibility                                                                 | Port  |
|--------------------------|--------------------------------------------------------------------------------|-------|
| **eureka-server**        | Service discovery (Netflix Eureka)                                             | 8761  |
| **weather-gateway**      | Entry point: aggregates data from forecast & alert, applies resilience patterns| 8080  |
| **forecast-service**     | Simulates external weather API (with configurable failure/delay)               | 8081  |
| **alert-service**        | Simulates weather alert service (rate-limited, occasional failures)           | 8082  |

All external calls from gateway are protected by Resilience4j.

## Tech Stack

- Spring Boot 3.x
- Resilience4j (CircuitBreaker, TimeLimiter, Retry, RateLimiter, Bulkhead)
- Spring Boot Actuator + Micrometer (resilience metrics)
- Spring Cloud OpenFeign (with Resilience4j integration)
- Spring Cloud Circuit Breaker (Resilience4j factory)
- Spring Cloud Netflix Eureka
- Lombok
- Maven (multi-module)
- Docker & Docker Compose

## Docker Containers

```yaml
services:
  eureka-server:
    build: ./eureka-server
    ports:
      - "8761:8761"

  weather-gateway:
    build: ./weather-gateway
    depends_on:
      - eureka-server
    ports:
      - "8080:8080"

  forecast-service:
    build: ./forecast-service
    depends_on:
      - eureka-server
    ports:
      - "8081:8081"
    environment:
      - SIMULATE_FAILURE_RATE=0.3  # 30% failures
      - SIMULATE_DELAY_MS=5000    # 5s delay

  alert-service:
    build: ./alert-service
    depends_on:
      - eureka-server
    ports:
      - "8082:8082"
    environment:
      - RATE_LIMIT_PER_MINUTE=10
```

Run with: `docker-compose up --build`

## Resilience4j Patterns Implemented

| Pattern                  | Configuration Details                                                  |
|--------------------------|------------------------------------------------------------------------|
| **Circuit Breaker**      | Failure rate threshold 50%, wait duration 30s, ring buffer size 100    |
| **Time Limiter**         | Timeout 3s on slow calls                                               |
| **Retry**                | 3 attempts, exponential backoff                                        |
| **Rate Limiter**         | 10 calls per 60s on alert service                                      |
| **Bulkhead**             | Semaphore (10 concurrent) or ThreadPool (20 threads)                    |
| **Fallback**             | Return cached/default data on failure                                  |
| **Shared State**         | Instance-level or shared via Redis (demo)                               |

## Key Features

- Declarative Resilience4j with @CircuitBreaker, @Retry, @TimeLimiter, @RateLimiter
- Programmatic API for dynamic policies
- Fallback methods with business-friendly defaults
- Metrics exposed via Actuator (/actuator/metrics/resilience4j.*)
- Circuit breaker state visualization
- Configurable failure/delay simulation in downstream services
- Shared circuit breaker state (optional Redis)
- Graceful degradation with cached responses

## Expected Endpoints

### Weather Gateway (`http://localhost:8080`)

| Method | Endpoint                        | Description                                      |
|--------|---------------------------------|--------------------------------------------------|
| GET    | `/api/weather/forecast`         | Get forecast → protected call to forecast-service|
| GET    | `/api/weather/alerts`           | Get alerts → rate limited + circuit breaker      |
| GET    | `/api/weather/dashboard`        | Aggregated view (combines both)                  |

### Downstream Services

- Forecast Service: `/api/forecast` — configurable delay/failure
- Alert Service: `/api/alerts` — rate limiting simulation

### Actuator (Gateway)

| Endpoint                                      | Description                                      |
|-----------------------------------------------|--------------------------------------------------|
| GET `/actuator/metrics/resilience4j.circuitbreaker.state` | CB state (OPEN/CLOSED/HALF_OPEN)        |
| GET `/actuator/metrics/resilience4j.circuitbreaker.failures` | Failure counts                        |

## Architecture Overview

```
Clients
   ↓
Weather Gateway
   ↓ (Resilience4j protected)
┌─────────────────────┬──────────────────────┐
│ forecast-service    │ alert-service        │
│ (unreliable)        │ (rate-limited)       │
└─────────────────────┴──────────────────────┘
   ↓ (on failure)
Fallback / Cached Response
   ↓
Metrics → Prometheus / Actuator
```

**Resilience Flow**:
1. Request → @CircuitBreaker("forecast")
2. Success → normal response
3. Failures exceed threshold → OPEN → fallback
4. After wait duration → HALF_OPEN → test calls
5. Success → CLOSED again

## How to Run

1. Clone repository
2. Start Docker: `docker-compose up --build`
3. Access Eureka: `http://localhost:8761`
4. Normal calls: `curl http://localhost:8080/api/weather/forecast`
5. Simulate high failure → circuit opens → fallback response
6. Check metrics: `/actuator/metrics/resilience4j.circuitbreaker.state{...,name="forecast"}`
7. Reduce failure rate → circuit closes automatically

## Testing Resilience

1. Normal → fast response
2. High failure in forecast-service → after threshold, fallback triggered
3. Circuit OPEN → all calls fast fallback
4. Wait 30s → HALF_OPEN → first call tests → closes if success
5. Rate limit alerts → 429 after limit
6. Concurrent calls → bulkhead limits parallelism

## Skills Demonstrated

- Advanced Resilience4j configuration (declarative + programmatic)
- Circuit breaker state management and fallbacks
- Combining multiple resilience patterns
- Metrics and observability of resilience events
- Protecting against cascading failures
- Graceful degradation strategies
- Production-ready external integration patterns

## Future Extensions

- Shared circuit breaker state with Redis
- Dynamic policy updates via config
- Integration with Spring Cloud Circuit Breaker
- Distributed tracing of fallback calls
- Custom events and listeners
- Dashboard for circuit breaker states
- Integration with Istio for hybrid resilience

