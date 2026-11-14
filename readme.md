# Backend Performance Best Practices

This checklist converts backend performance best practices into actionable rules for an AI coding agent, ordered from highest-leverage (data layer) to longest-term operational practices.

Source: roadmap.sh/backend-performance-best-practices

---

## 1. Caching Strategy

### Caching Mechanisms

* **Use the right layer for the data**

  * HTTP caching (`Cache-Control`, `ETag`)
  * Server/client-side caching (Redis, Memcached, in-memory)
  * CDN caching for static/frequently-requested content

---

### Caching Patterns

Choose deliberately, not by default:

* **Cache-aside** — check cache → miss → fetch from source → populate cache
* **Write-through** — write to cache and source together
* **Read-through** — cache layer loads missing data itself
* **Write-behind (write-back)** — write to cache immediately, persist to source asynchronously. Faster writes, but risks data loss on cache failure before the write lands — only use when that tradeoff is acceptable

---

### Cache Invalidation

* Never cache without an invalidation strategy.
* Use one of:

  * TTL expiry
  * Explicit purge-on-write
  * Versioned cache keys
* Guard against **cache stampede** (thundering herd) — when a hot key expires, don't let every concurrent request miss the cache and hit the source simultaneously. Use one of:

  * Request coalescing / locking (only one request repopulates, others wait)
  * Early/probabilistic refresh before TTL expiry
  * Stale-while-revalidate (serve stale data while refreshing in the background)

---

## 2. Database Performance

### Connection Management

* **Use connection pooling.** Never open/close a raw connection per request.
* Tune pool settings explicitly:

```text
max connections
idle timeout
connection reuse params
```

---

### Query & Index Optimization

* Index columns that are filtered, sorted, or joined on.
* Audit ORM-generated queries — eliminate N+1 patterns.
* Never use:

```sql
SELECT *
```

Fetch only the columns the code actually uses.

---

### Data Retrieval Patterns

* Choose deliberately per access pattern:

  * Lazy loading
  * Eager loading
  * Batch processing
* Paginate large result sets. Never return or load an entire table at once.

---

### Schema & Maintenance

* Denormalize for read-heavy workloads when it reduces costly JOINs.
* Minimize and optimize JOINs — remove any that aren't necessary.
* Run regular maintenance:

```text
Vacuum
Re-index
Clean up unused/stale data
```

* Enable slow-query logging and monitor it — treat a slow query as a bug.

---

### Scaling & Diagnostics

* Set up **replication** for redundancy and to offload reads.
* **Shard** when a single instance can't handle the load.
* Use the database engine's own profiling tools before guessing at bottlenecks.

---

## 3. Network Optimization

### Latency Reduction

* Host the backend close to its users (regional/edge deployment).

---

### Connection Reuse

* Use **HTTP keep-alive** to reuse connections instead of reopening per request.

---

### Content Delivery

* Serve static and frequently-accessed assets via a **CDN**, not the origin server.
* Prefetch or preload resources/data the next request will need.

---

## 4. API Response Optimization

### Payload Control

* Enforce payload size limits on requests and responses.

```text
Example:
JSON body: 1MB
File upload: configurable, explicit max
```

* Enable compression (`gzip`/`brotli`) on responses.
* Paginate large datasets in API responses.
* Support sparse fieldsets — let clients request only the fields they need (`?fields=id,name`) instead of always returning the full object.

---

### Processing Efficiency

* Cut unnecessary processing or expensive computation out of the request path.
* Move anything non-essential to the response out of the synchronous flow (see §5).

---

## 5. Asynchronous Processing

### Background Jobs

* Offload heavy or slow tasks to background jobs/queues.
* Never block the request/response cycle on:

```text
Email sending
Image/video processing
Report generation
Bulk exports
```

---

### Message Brokers

* Use message brokers (RabbitMQ, Kafka, etc.) for async communication between services instead of tight synchronous coupling.

---

## 6. Load Balancing & Scaling

### Scaling Strategy

* **Vertical scaling** — more CPU/RAM on a single instance.
* **Horizontal scaling** — more instances sharing the load.
* Choose based on what the workload actually needs — don't default to one.

---

### Traffic Distribution

* Distribute traffic across instances with a load balancer.
* Never let a single instance become a single point of failure.

---

## 7. Code-Level Optimization

### Profiling

* Profile before optimizing. Don't guess at bottlenecks.

---

### Algorithmic Efficiency

* Choose algorithms and data structures deliberately for the task, not by habit.

---

### Critical Path Prioritization

* Prioritize optimization effort on critical paths and frequently-hit endpoints — that's where it has outsized impact.

---

### Language Choice

* Consider a compiled language (Go, Rust) for performance-critical segments.

---

### Architectural Decomposition

* Evaluate SOA/microservices decomposition only when a monolith's scaling limits are actually being hit — not preemptively.

---

### Resilience Under Load

* Set explicit connection timeouts.
* Implement efficient retry logic for network calls (backoff, max attempts).
* Use a **circuit breaker** around calls to unreliable or overloaded dependencies — stop sending requests to a failing dependency for a cooldown period instead of retrying into it and compounding load.

  * States: closed (normal) → open (fail fast, no calls) → half-open (test with limited traffic) → closed or open again
* Design retries to be idempotent — a retried write must not duplicate the effect of the original.

---

### Request Batching

* Batch similar requests together to cut round-trip overhead.

---

## 8. Security Practices Affecting Performance

### Dependency Hygiene

* Keep dependencies up to date.

---

### Access Control

* Enforce proper authentication and authorization on every access path.

---

### Rate Limiting

* Apply request throttling and rate limiting to protect backend resources from abuse/overload.

```text
Example:
100 requests/min per user
1000 requests/min per IP
```

---

### Security Audits

* Audit and update security measures on a recurring basis, not just once.

---

## 9. Monitoring and Logging

### Observability Stack

* Instrument comprehensive monitoring and logging for performance metrics and troubleshooting — don't ship blind.
* Use standard tooling rather than ad hoc solutions:

```text
Metrics: Prometheus
Dashboards: Grafana
Logs: ELK stack
```

---

### Async Logging

* Log asynchronously so logging never blocks the request-handling path.

---

## 10. Performance Testing

### Benchmarking Cadence

* Run performance testing and benchmarking regularly, not just once at launch.
* Use it to:

```text
Catch performance regressions
Track improvements over time
Validate optimization efforts
```

---

## Result

This checklist covers the full performance surface of a backend system:

| Layer | Covered |
| --- | --- |
| Data layer (caching/DB) | ✔ |
| Network layer | ✔ |
| Application layer | ✔ |
| Scaling & resilience | ✔ |
| Operational (monitoring/testing) | ✔ |
