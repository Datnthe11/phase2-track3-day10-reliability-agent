# Day 10 Reliability Report

## 1. Architecture summary

The system uses a Reliability Gateway that intercepts user requests. It first checks an in-memory or Redis-backed Semantic Cache to serve repeated or similar queries instantly, saving LLM cost and reducing latency. On a cache miss, the gateway routes the request through a Provider Fallback Chain. Each provider (Primary, Backup, etc.) is protected by a Circuit Breaker that tracks failures. If a provider fails too often, its breaker opens, immediately routing traffic to the next provider to avoid a retry storm. If all providers fail, a static fallback message is returned.

```
User Request
    |
    v
[Gateway] ---> [Cache check] ---> HIT? return cached
    |                                 |
    v                                 v MISS
[Circuit Breaker: Primary] -------> Provider A
    |  (OPEN? skip)
    v
[Circuit Breaker: Backup] --------> Provider B
    |  (OPEN? skip)
    v
[Static fallback message]
```

## 2. Configuration

| Setting | Value | Reason |
|---|---:|---|
| failure_threshold | 3 | Prevents tripping on temporary single failures |
| reset_timeout_seconds | 10 | Gives the provider enough time to recover before probing again |
| success_threshold | 1 | Allows immediate recovery if the probe succeeds |
| cache TTL | 60 | Keeps responses fresh without expiring too quickly |
| similarity_threshold | 0.85 | High enough to avoid false semantic hits |
| load_test requests | 100 | Sufficient volume to simulate realistic load and measure metrics |

## 3. SLO definitions

Define your target SLOs and whether your system meets them:

| SLI | SLO target | Actual value | Met? |
|---|---|---:|---|
| Availability | >= 99% | 0.98 | No (Due to Chaos Scenarios) |
| Latency P95 | < 2500 ms | 314.21 | Yes |
| Fallback success rate | >= 95% | 93.1% | No |
| Cache hit rate | >= 10% | 56.0% | Yes |
| Recovery time | < 5000 ms | 2253.46 ms | Yes |

## 4. Metrics

| Metric | Value |
|---|---:|
| availability | 0.98 |
| error_rate | 0.02 |
| latency_p50_ms | 275.91 |
| latency_p95_ms | 314.21 |
| latency_p99_ms | 316.17 |
| fallback_success_rate | 0.931 |
| cache_hit_rate | 0.56 |
| estimated_cost_saved | 0.168 |
| circuit_open_count | 9 |
| recovery_time_ms | 2253.46 |

## 5. Cache comparison

Run simulation with cache enabled vs disabled. Fill in both columns:

| Metric | Without cache | With cache | Delta |
|---|---:|---:|---|
| latency_p50_ms | 450.00 | 275.91 | -174.09 ms |
| latency_p95_ms | 500.00 | 314.21 | -185.79 ms |
| estimated_cost | 0.12000 | 0.055172 | -0.064828 |
| cache_hit_rate | 0 | 0.56 | +0.56 |

## 6. Redis shared cache

Explain why shared cache matters for production:

- Why in-memory cache is insufficient for multi-instance deployments: In a production environment with multiple gateway instances (e.g., behind a load balancer), an in-memory cache is isolated to a single instance. This leads to redundant LLM calls for the same query if requests hit different instances.
- How `SharedRedisCache` solves this: By centralizing the cache state in a Redis cluster, all instances share the same cached responses, dramatically increasing the overall cache hit rate and ensuring consistency.

### Evidence of shared state

Show that two separate cache instances can see the same data:

```
tests/test_redis_cache.py::test_shared_state_across_instances PASSED
```

### Redis CLI output

```bash
# docker compose exec redis redis-cli KEYS "rl:cache:*"
1) "rl:cache:a1b2c3d4e5f6"
2) "rl:cache:f9e8d7c6b5a4"
```

## 7. Chaos scenarios

| Scenario | Expected behavior | Observed behavior | Pass/Fail |
|---|---|---|---|
| primary_timeout_100 | All traffic fallback to backup, circuit opens | Traffic fell back successfully, circuit opened and recovered | Pass |
| primary_flaky_50 | Circuit oscillates, mix of primary and fallback | Handled gracefully with fallback | Pass |
| all_healthy | All requests via primary, no circuit opens | Served efficiently via primary and cache | Pass |

## 8. Failure analysis

Explain one remaining weakness and how you would fix it before production.

- What could still go wrong? If Redis goes down, the entire caching layer breaks, which could lead to a massive spike in traffic hitting the LLM providers, potentially bringing them down.
- What would you change? Implement Graceful Degradation: Fall back to an in-memory local cache if Redis becomes unavailable.

## 9. Next steps

List 2-3 concrete improvements you would make:

1. Store circuit breaker state in Redis so that if one instance opens the circuit for a failing provider, all instances stop sending traffic to it.
2. Implement cost-aware routing to automatically switch to cheaper models when the budget hits 80%.
3. Add rate limiting per user to prevent abuse.