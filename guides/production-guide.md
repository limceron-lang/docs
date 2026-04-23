# Limceron Production Guide

## Designing for Fault Tolerance

Limceron is built with the assumption that **everything fails in production**.
The language and stdlib enforce patterns that make failure handling explicit,
recoverable, and observable.

### Principle 1: No Panics in Library Code

The stdlib never panics. Every operation that can fail returns `Result<T>`.
Your application code should follow the same pattern.

```limceron
// BAD: this will abort if the file doesn't exist
let content = fs.read_file("config.json").unwrap()

// GOOD: handle the error
let content = fs.read_file("config.json")?

// GOOD: provide a fallback
let content = fs.read_file("config.json")
    .unwrap_or(default_config)
```

### Principle 2: Graceful Degradation

```limceron
use async
use http
use log

fn main() -> Result<void> {
    let server = http.Server.new()

    // Setup routes...

    // Graceful shutdown: finish in-flight requests before stopping
    let shutdown = async.signal(os.SIGTERM, os.SIGINT)

    spawn {
        await shutdown
        log.info("shutdown signal received, draining connections...")
        server.shutdown(timeout: 30.seconds())
    }

    server.listen(":8080")
}
```

### Principle 3: Timeouts on Everything

```limceron
// Every external call should have a timeout
let resp = http.get("https://api.example.com/data")
    .timeout(5.seconds())?

// Database queries too
let users = pool.query<User>("SELECT * FROM users")
    .timeout(10.seconds())?

// Or use a context with deadline
let ctx = Context.with_timeout(30.seconds())
let result = do_complex_work(ctx)?
```

### Principle 4: Retry with Backoff

```limceron
use async.{retry, RetryConfig}

let config = RetryConfig {
    max_attempts: 3,
    initial_delay: 100.milliseconds(),
    max_delay: 5.seconds(),
    backoff: Backoff.Exponential(factor: 2.0),
    retry_on: |err| err.is_transient(),  // only retry transient errors
}

let result = retry(config, || {
    external_api.call(request)
})?
```

### Principle 5: Circuit Breakers

```limceron
use async.CircuitBreaker

let breaker = CircuitBreaker.new(
    failure_threshold: 5,        // open after 5 failures
    reset_timeout: 30.seconds(), // try again after 30s
    half_open_max: 3,            // allow 3 test requests when half-open
)

fn call_service(req: Request) -> Result<Response> {
    breaker.execute(|| {
        http.post("https://payment-service/charge", req)?
    })
}
```

### Principle 6: Structured Concurrency

```limceron
use async.{TaskGroup, Supervisor}

// TaskGroup: all tasks must succeed, or all are cancelled
fn process_batch(items: &[Item]) -> Result<Vec<Output>> {
    let group = TaskGroup.new()

    for item in items {
        group.spawn(|| process_item(item))
    }

    group.wait()  // returns Err if ANY task fails
}

// Supervisor: restart failed tasks automatically
fn run_workers() -> Result<void> {
    let sup = Supervisor.new(strategy: RestartStrategy.OneForOne)

    sup.add("ingester", || run_ingester())
    sup.add("processor", || run_processor())
    sup.add("exporter", || run_exporter())

    sup.start()  // restarts any worker that fails
}
```

---

## Observability

### Structured Logging
```limceron
use log

fn handle_request(req: &Request) -> Result<Response> {
    let start = time.Instant.now()

    log.info("request started",
        method: req.method,
        path: req.path,
        request_id: req.id,
    )

    let result = process(req)

    let elapsed = start.elapsed()
    match &result {
        Ok(_) -> log.info("request complete",
            method: req.method,
            path: req.path,
            status: 200,
            latency_ms: elapsed.as_millis(),
        )
        Err(e) -> log.error("request failed",
            method: req.method,
            path: req.path,
            error: e.message(),
            latency_ms: elapsed.as_millis(),
        )
    }

    result
}
```

### Health Checks
```limceron
use http

interface HealthChecker {
    fn name(&self) -> string
    fn check(&self) -> Result<void>
}

struct AppHealth {
    checks: Vec<Box<HealthChecker>>
}

impl AppHealth {
    fn add(&mut self, check: impl HealthChecker) {
        self.checks.push(Box.new(check))
    }

    fn handler(&self) -> http.Handler {
        |req| {
            let mut results = #{}
            let mut healthy = true

            for check in &self.checks {
                match check.check() {
                    Ok(_) -> results[check.name()] = "ok"
                    Err(e) -> {
                        results[check.name()] = e.message()
                        healthy = false
                    }
                }
            }

            let status = if healthy { 200 } else { 503 }
            Ok(http.Response.json(results).status(status))
        }
    }
}
```

---

## Deployment

### Static Binary
Limceron compiles to a single static binary by default. No shared library dependencies.

```bash
limceron build --release
# Output: ./my-service (single binary, statically linked)

# Check: no dynamic dependencies
ldd ./my-service
# not a dynamic executable
```

### Docker (Scratch Image)
```dockerfile
# Build stage
FROM limceron:latest AS builder
COPY . /src
WORKDIR /src
RUN limceron build --release --target=linux-x86_64

# Runtime stage — empty base image
FROM scratch
COPY --from=builder /src/my-service /my-service
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
EXPOSE 8080
ENTRYPOINT ["/my-service"]
```

Image size: ~5-15 MB (just your binary + TLS certs).

### Cross-Compilation
```bash
# Build for all targets
limceron build --release --target=linux-x86_64
limceron build --release --target=linux-arm64
limceron build --release --target=darwin-arm64
limceron build --release --target=windows-x86_64
```

### Reproducible Builds
```bash
# Deterministic output — same source = same binary
limceron build --release --reproducible

# Verify
sha256sum ./my-service
# Same hash every time, on any machine
```

---

## Performance

### Release Builds
```bash
limceron build --release          # optimizations enabled
limceron build --release --lto    # link-time optimization (slower build, faster binary)
limceron build --release --pgo    # profile-guided optimization (requires profiling data)
```

### Profiling
```bash
limceron build --release --profile
./my-service                  # run under load
limceron profile analyze ./my-service.prof
```

### Memory Efficiency
```limceron
// Use arena allocators for batch processing
fn process_requests(requests: &[Request]) -> Result<Vec<Response>> {
    let arena = Arena.new(capacity: 64.kb())
    let mut responses = Vec.with_capacity(requests.len())

    for req in requests {
        arena.reset()  // reuse memory, zero allocations
        let temp = arena.alloc<TempData>()
        // ... process with temp ...
        responses.push(build_response(temp))
    }

    Ok(responses)
}
```

---

## Security Checklist

- [ ] All user input validated at system boundaries
- [ ] SQL queries use parameterized statements (never string concatenation)
- [ ] TLS enabled for all network connections
- [ ] Secrets loaded from environment variables, never hardcoded
- [ ] `unsafe` blocks minimized and audited (`limceron audit unsafe`)
- [ ] Dependencies pinned with checksums (`limceron deps verify`)
- [ ] Binary compiled with `--release` (includes stack canaries, ASLR)
- [ ] Logging does not include sensitive data (passwords, tokens, PII)
- [ ] Rate limiting on all public endpoints
- [ ] Graceful shutdown handles SIGTERM
