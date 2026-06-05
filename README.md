# PRD: Concurrency Benchmark Platform
## Multi-Language HTTP Server Load Testing & Metrics Dashboard

---

## **1. Overview**

Build a comparative benchmarking platform that measures how different programming languages handle **1000 concurrent requests per second** in pure implementations (no external message queues, caching layers, or load balancers).

Each language gets a simple HTTP server that:
- Accepts concurrent POST requests
- Performs minimal I/O (simulating real work)
- Reports metrics in real-time
- Runs in isolated Docker containers

A single **shared HTML dashboard** visualizes metrics across all running servers via WebSocket or polling.

---

## **2. Objectives**

1. **Learn concurrency models** across Python, Go, Java, Node.js, Rust, C# (.NET)
2. **Measure real-world performance** under sustained load (not peak burst)
3. **Identify bottlenecks** per language (GIL, thread pool limits, context switching, memory)
4. **Practical tooling** — use Docker to isolate, load generation that's repeatable

---

## **3. Scope**

### **In Scope**
- HTTP server implementations in 6+ languages
- Load generator (simple, separate)
- Real-time metrics collection (latency, throughput, errors, memory)
- Web UI dashboard with live graphs
- Docker Compose orchestration (spin up all servers + dashboard at once)

### **Out of Scope**
- Database connections (keep servers stateless)
- Authentication/authorization
- Distributed tracing (too complex for this)
- Kubernetes/production deployments
- Load balancing or reverse proxy

---

## **4. Functional Requirements**

### **4.1 HTTP Server (Language-Agnostic)**

Each language implementation must:

**Request Handler:**
```
POST /request
Content-Type: application/json

{
  "operation": "cpu" | "io" | "mixed",
  "duration_ms": 10,
  "payload_size_kb": 1
}

Response:
{
  "status": "ok",
  "request_id": "uuid",
  "processed_at_ms": 42
}
```

**Operations:**
- `cpu`: Busy-loop for N milliseconds (tests thread parallelism)
- `io`: Simulate I/O wait (sleep, non-blocking if language supports)
- `mixed`: 50% CPU, 50% I/O

**Metrics Endpoint (Server → Dashboard):**
```
GET /metrics

{
  "timestamp": 1717548000,
  "requests_total": 50000,
  "requests_success": 49995,
  "requests_failed": 5,
  "requests_per_sec": 1020,
  "p50_latency_ms": 8,
  "p95_latency_ms": 25,
  "p99_latency_ms": 150,
  "memory_mb": 256,
  "cpu_percent": 85,
  "active_connections": 1000
}
```

**Health Check:**
```
GET /health → 200 OK
```

### **4.2 Load Generator**

Standalone tool (Python or Go) that:
- Maintains 1000 concurrent connections
- Sends requests at configurable rate (e.g., 1000 RPS)
- Rotates through operation types (cpu, io, mixed)
- Adjustable payload size
- Runs for N minutes
- Can target multiple servers in parallel

**CLI:**
```bash
./load-gen \
  --target http://python-server:5000 \
  --connections 1000 \
  --rps 1000 \
  --duration 300 \
  --operation cpu \
  --duration-ms 10
```

### **4.3 Dashboard (Web UI)**

**Single HTML file** with embedded JavaScript, connects to all running servers.

**Displays:**
- **Live graphs** (update every 1 sec):
  - Throughput over time (requests/sec per language)
  - Latency percentiles (p50, p95, p99)
  - Memory usage (line chart)
  - CPU utilization (stacked or separate)
  - Error rate (spike detection)

- **Comparison table:**
  - Language, framework version
  - Current RPS, avg latency, p99
  - Total requests, success rate
  - Memory peak, CPU peak
  - Status (running, stopped)

- **Controls:**
  - Start/stop load test
  - Choose operation type (cpu, io, mixed)
  - Adjust RPS target
  - Export metrics as CSV
  - Clear history

---

## **5. Non-Functional Requirements**

### **5.1 Performance Targets**

| Language | Target RPS | Target P99 Latency | Memory Limit |
|----------|------------|-------------------|--------------|
| Python (asyncio) | 800–1200 | <100ms | 512MB |
| Go | 2000+ | <50ms | 128MB |
| Java (virtual threads) | 1500–2000 | <100ms | 512MB |
| Node.js | 1000–1500 | <75ms | 256MB |
| Rust (tokio) | 2500+ | <30ms | 64MB |
| C# (.NET 8) | 1500+ | <75ms | 256MB |

*(These are educated guesses; benchmark will prove actual capability)*

### **5.2 Reliability**
- No server crashes under sustained 1000 RPS load
- <0.1% error rate (dropped/timeout requests)
- Memory should stabilize, not leak

### **5.3 Observability**
- Server logs requests (debug mode, minimal overhead)
- Metrics available every 1 second
- Dashboard updates with <2 second latency

### **5.4 Development**
- Reproducible: same load generator, same operation, produces consistent results
- Isolated: each language runs in own Docker container
- Comparable: same request/response format across all servers

---

## **6. Technical Architecture**

### **6.1 Container Structure**

```
docker-compose.yml
├── python-server (port 5001)
├── go-server (port 5002)
├── java-server (port 5003)
├── node-server (port 5004)
├── rust-server (port 5005)
├── csharp-server (port 5006)
├── load-generator (one-shot, targets all servers)
└── dashboard (port 8080, static HTML + JS)
```

### **6.2 Communication**

**Server → Dashboard:**
- Metrics endpoint `/metrics` (HTTP GET)
- Dashboard polls each server every 1 second OR
- Dashboard opens WebSocket if servers support it (nicer UX)

**Load Generator → Server:**
- HTTP POST to `/request`
- Maintain connection pool

**Dashboard → Load Gen:**
- Docker Compose passes environment (server URLs)
- Dashboard is static HTML, shows live data only if metrics available

### **6.3 Metrics Collection**

**In-Server (not exported):**
- Request counter (atomic)
- Latency histogram (sliding window of recent 1000 requests)
- Success/failure count
- Memory tracking (process RSS)
- CPU utilization (if available)

**Per-Second Snapshot:**
- Aggregate histogram buckets → p50, p95, p99
- Calculate RPS from counter delta
- Read OS metrics (via /proc on Linux, system calls on macOS/Windows)

---

## **7. Implementation Details by Language**

### **7.1 Python (asyncio)**

```python
# Framework: aiohttp
# Concurrency: asyncio event loop
# Threading: single-threaded, event-driven

Key points:
- aiohttp handles concurrent requests
- GIL: only affects sync CPU work; asyncio runs on single thread (no GIL contention)
- For CPU operation: offload to ThreadPoolExecutor (will be slow due to GIL)
- Memory: lightweight coroutines
```

**Expected behavior:**
- Handles I/O well (async sleep simulates I/O)
- CPU work causes GIL bottleneck
- Likely plateaus at 800–1200 RPS

---

### **7.2 Go**

```go
// Net/HTTP package + goroutines
// Concurrency: goroutines (M:N threading)
// Scheduler: runtime scheduler multiplexes goroutines onto OS threads

Key points:
- net/http spins goroutine per connection
- Thousands of goroutines cheap (~1KB each)
- No GIL: true parallelism on multi-core
- CPU work: goroutine runs on available core
```

**Expected behavior:**
- Scales to 2000+ RPS
- Low memory, fast startup
- Best-in-class latency

---

### **7.3 Java (Virtual Threads)**

```java
// Framework: Spring Boot / Undertow
// Concurrency: Virtual Threads (Project Loom, Java 21+)
// Threads: 1:1 virtual-to-kernel thread mapping

Key points:
- Virtual threads are cheap (like goroutines)
- JVM warmup: takes ~10 seconds to reach peak performance
- GC pauses can spike latency
- CPU work: true parallelism
```

**Expected behavior:**
- 1500–2000 RPS (after warmup)
- Memory higher than Go initially (JVM overhead)
- Potential GC pause spikes in p99 latency

---

### **7.4 Node.js**

```javascript
// Framework: Express or raw http module
// Concurrency: event loop + libuv thread pool
// Model: single-threaded event loop, blocking I/O offloaded to thread pool

Key points:
- Event loop is single-threaded (no true parallelism)
- I/O (sleep) is non-blocking → good throughput
- CPU work (busy-loop) blocks event loop → request queuing
- Thread pool default: 4 threads (configurable)
```

**Expected behavior:**
- Good for I/O (1000+ RPS on async sleep)
- CPU work causes bottleneck (event loop starvation)
- Memory efficient for I/O workloads

---

### **7.5 Rust (tokio)**

```rust
// Async runtime: tokio
// Concurrency: async/await + tasks
// Model: M:N threading, works-stealing scheduler

Key points:
- tokio is highly optimized
- Zero-cost abstractions (no GC, no runtime overhead)
- True parallelism on multi-core
- Borrow checker ensures memory safety
```

**Expected behavior:**
- 2500+ RPS easily
- Lowest memory footprint
- Lowest latency

---

### **7.6 C# / .NET 8**

```csharp
// Framework: ASP.NET Core minimal APIs
// Concurrency: async/await on thread pool
// Model: thread pool + async I/O

Key points:
- Async/await is built-in (modern)
- Thread pool scales automatically
- GC can cause latency spikes
- JIT compilation (startup time)
```

**Expected behavior:**
- 1500+ RPS
- Good balance of performance and memory
- Consistent latency (mature GC)

---

## **8. Data & Metrics**

### **Per-Server State (in-memory)**
```json
{
  "language": "go",
  "start_time": 1717548000,
  "metrics_window": [
    {
      "timestamp": 1717548001,
      "requests_total": 1000,
      "requests_success": 999,
      "requests_failed": 1,
      "latencies_ms": [5, 8, 12, 15, 25, 150],  // all requests in window
      "memory_mb": 45,
      "cpu_percent": 78
    },
    // ... more windows
  ]
}
```

### **Dashboard State**
```json
{
  "servers": {
    "python": { ... },
    "go": { ... },
    "java": { ... },
    "node": { ... },
    "rust": { ... },
    "csharp": { ... }
  },
  "load_test": {
    "status": "running",
    "start_time": 1717548000,
    "operation": "cpu",
    "target_rps": 1000
  }
}
```

---

## **9. Success Metrics**

1. **Reproducibility:** Run same test 3 times, results vary <10%
2. **No crashes:** 5-minute sustained 1000 RPS load, zero crashes
3. **Accuracy:** Metrics match independent load test tool (e.g., wrk, hey)
4. **Usability:** Dashboard loads instantly, updates smoothly
5. **Learning:** Clear documentation on why each language behaves differently

---

## **10. Deliverables**

### **Phase 1: Core (Week 1–2)**
- [ ] Python asyncio server
- [ ] Go server
- [ ] Load generator (Go or Python)
- [ ] Basic metrics collection

### **Phase 2: Expansion (Week 2–3)**
- [ ] Java virtual threads server
- [ ] Node.js server
- [ ] Rust tokio server
- [ ] C# .NET server

### **Phase 3: UI & Polish (Week 3–4)**
- [ ] HTML dashboard with live graphs (Chart.js or similar)
- [ ] Docker Compose orchestration
- [ ] Documentation per language (why it performs as it does)
- [ ] Benchmark results, writeup

### **Final Deliverable**
```
concurrency-benchmark/
├── README.md (setup, how to run, results)
├── docker-compose.yml
├── load-gen/ (load generator source)
├── servers/
│   ├── python/
│   ├── go/
│   ├── java/
│   ├── node/
│   ├── rust/
│   └── csharp/
├── dashboard/ (HTML + JS)
└── results/ (CSV exports, graphs, writeup)
```

---

## **11. Key Questions to Answer During Build**

1. **Python GIL:** How much does async/await help vs. multiprocessing?
2. **Go scheduling:** Does GOMAXPROCS tuning improve RPS?
3. **Java warmup:** How long until peak performance?
4. **Node.js CPU:** Can we work around event loop blocking?
5. **Rust zero-cost:** How much better than Java/Go?
6. **Memory leaks:** Any language leak under sustained load?
7. **Latency tail:** Which language has the worst p99?
8. **Scalability:** Where does each language plateau?

---

## **12. Out-of-Scope But Interesting**

- **Scaling across machines** (distributed load testing)
- **TLS/HTTPS overhead** (add if curious)
- **Database connections** (would confound results)
- **Custom kernel tuning** (ulimit, sysctl)
- **Exotic languages** (Elixir, Haskell, Zig—add if motivated)

---

## **13. Success Criteria**

- **All servers handle 1000 RPS without crashing**
- **Dashboard shows accurate, real-time metrics**
- **Clear, reproducible results**
- **Documented explanation for each language's behavior**

---

## **14. Timeline**

| Phase | Duration | Milestones |
|-------|----------|-----------|
| Setup & Python/Go | 1 week | Load gen working, Python & Go hitting 1000 RPS |
| Java/Node/Rust/C# | 1 week | All 6 servers running, metrics flowing |
| Dashboard & Polish | 1 week | Live UI, Docker Compose, docs |
| Results & Analysis | 1 week | Benchmarks complete, writeup done |

**Total:** ~4 weeks part-time

---

## **15. Tech Stack Summary**

| Component | Tech |
|-----------|------|
| Load Gen | Python (locust) OR Go (custom) |
| Python Server | aiohttp |
| Go Server | net/http + goroutines |
| Java Server | Spring Boot + Virtual Threads (Java 21+) |
| Node Server | Express or http module |
| Rust Server | tokio + axum |
| C# Server | ASP.NET Core |
| Dashboard | HTML5 + Chart.js + vanilla JS |
| Orchestration | Docker Compose |
| Metrics | In-memory snapshots (no external DB) |

---

## **16. Risks & Mitigation**

| Risk | Mitigation |
|------|-----------|
| Java JVM warmup skews results | Pre-warm JVM before load test starts |
| Load generator is bottleneck | Use multiple load gen instances or Go-based tool |
| Dashboard polling overwhelms servers | Cache metrics locally, poll every 1-2 seconds |
| GC pauses cause outlier latencies | Use low-pause GC algorithms (ZGC, Shenandoah) |
| Network is bottleneck | Run everything on localhost (Docker) |
| Metrics collection overhead | Sample, don't track every request |

---

# 17. Benchmark Integrity & Scientific Methodology

This section supplements the benchmark design and defines rules that ensure results are fair, reproducible, and scientifically defensible.

---

## 17.1 Benchmark Fairness Rules

To ensure that differences observed are caused by language/runtime characteristics rather than implementation inconsistencies, all benchmark participants must follow the same rules.

### Infrastructure Rules

All servers must:

* Run on the same physical machine
* Use the same Docker network
* Use identical CPU allocations
* Use identical memory allocations
* Use identical operating system kernel
* Use identical benchmark duration
* Use identical warmup duration

### Application Rules

All implementations must:

* Expose identical endpoints
* Accept identical request payloads
* Return identical response payloads
* Use JSON serialization
* Avoid external dependencies that affect performance
* Remain stateless
* Avoid caching request results

### Prohibited Optimizations

The following are not allowed:

* Redis caching
* External message queues
* Database-backed request processing
* Reverse proxies
* Request batching
* Custom kernel tuning
* Language-specific shortcuts that alter benchmark semantics

The objective is to compare concurrency models and runtime behavior, not infrastructure optimizations.

---

## 17.2 Resource Constraints

To ensure fairness, each container must operate under identical limits.

Example Docker configuration:

```yaml
deploy:
  resources:
    limits:
      cpus: "2"
      memory: 512M
```

Recommended baseline:

| Resource | Value                |
| -------- | -------------------- |
| CPU      | 2 vCPU               |
| Memory   | 512 MB               |
| Network  | Shared Docker bridge |
| Storage  | Ephemeral            |

The benchmark must document the host machine specifications alongside every published result.

---

## 17.3 Benchmark Modes

Different workloads test different runtime characteristics.

A single benchmark score is insufficient.

The platform will support three workload categories.

### Mode A — Concurrency Benchmark

Purpose:

Measure scheduler efficiency, async runtime behavior, and context-switch overhead.

Workload:

```text
sleep(10ms)
```

Examples:

* asyncio.sleep()
* tokio::sleep()
* Task.Delay()
* setTimeout()

Measures:

* Async runtime efficiency
* Scheduler scalability
* Event-loop behavior
* Connection handling

---

### Mode B — CPU Benchmark

Purpose:

Measure parallel execution capability and runtime efficiency.

Workload examples:

```text
SHA256 x 10000
```

or

```text
Fibonacci(35)
```

Requirements:

* Same algorithm
* Same implementation rules
* Same computational workload

Measures:

* Multi-core utilization
* Runtime overhead
* Scheduling efficiency
* Raw execution performance

---

### Mode C — Mixed Benchmark

Purpose:

Simulate realistic backend workloads.

Composition:

* 50% CPU work
* 50% asynchronous waiting

Measures:

* Runtime adaptability
* Thread pool behavior
* Scheduler effectiveness under mixed pressure

---

## 17.4 Benchmark Execution Methodology

Every benchmark run must follow the same lifecycle.

### Phase 1 — Warmup

Duration:

```text
60 seconds
```

Purpose:

* Trigger JIT compilation
* Allow runtime optimization
* Stabilize thread pools
* Populate caches

Warmup results are discarded.

---

### Phase 2 — Measurement

Duration:

```text
300 seconds
```

Purpose:

Collect benchmark data.

Metrics recorded:

* Throughput
* Latency
* Memory
* CPU utilization
* Error rate

---

### Phase 3 — Cooldown

Duration:

```text
30 seconds
```

Purpose:

Allow final metric collection and graceful shutdown.

---

### Phase 4 — Repeatability

Each benchmark must execute:

```text
3 independent runs
```

Final result:

```text
Median(run1, run2, run3)
```

Outliers are reported but not used as primary scores.

---

## 17.5 Latency Measurement Standard

Raw latency arrays are not retained indefinitely.

Instead, implementations must use histogram-based tracking.

Recommended options:

* HDR Histogram
* DDSketch
* Prometheus Histogram

Metrics reported:

* p50
* p90
* p95
* p99
* p99.9

Benefits:

* Lower memory usage
* Better tail-latency analysis
* More accurate percentile calculations

---

## 17.6 Validation Against External Tools

To ensure benchmark credibility, internal metrics must be validated against independent load-testing tools.

Recommended validators:

* wrk
* hey
* vegeta
* autocannon

Acceptable deviation:

| Metric     | Maximum Deviation |
| ---------- | ----------------- |
| Throughput | ±5%               |
| Latency    | ±5%               |
| Error Rate | ±1%               |

Any larger deviation must be investigated before publishing results.

---

## 17.7 Benchmark Report Format

Every benchmark publication must include:

### Environment

* Host CPU
* Core count
* RAM
* OS version
* Docker version

### Runtime Versions

* Python version
* Go version
* Java version
* Node.js version
* Rust version
* .NET version

### Workload

* Benchmark mode
* Target RPS
* Concurrent connections
* Duration

### Results

| Language | Avg RPS | p95 | p99 | CPU Peak | Memory Peak | Errors |
| -------- | ------- | --- | --- | -------- | ----------- | ------ |

### Analysis

For each language:

* Observed bottlenecks
* Scheduler behavior
* Runtime characteristics
* Unexpected findings

---

## 17.8 Interpretation Guidelines

Benchmark results must be interpreted cautiously.

Higher throughput does not necessarily indicate a better production platform.

The benchmark measures:

* Concurrency handling
* Runtime efficiency
* Resource utilization

The benchmark does not measure:

* Developer productivity
* Ecosystem maturity
* Framework quality
* Long-term maintainability
* Operational complexity

Results should be treated as workload-specific observations rather than universal rankings.

**END PRD**
