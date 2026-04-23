# Limceron Standard Library Overview

The standard library ships with the compiler and requires no external dependencies.
All stdlib packages are available without any `limceron deps add` — they're always there.

## Package Map

```
stdlib/
├── core         Primitives, traits, Result, Option, iterators, memory
├── collections  HashMap, BTreeMap, Vec, Set, Queue, Deque, RingBuffer
├── fmt          String formatting and interpolation
├── math         Math functions, BigInt, BigDecimal
├── encoding     Base64, hex, URL encoding, UTF conversions
│
├── io           Reader/Writer traits, buffered I/O, pipes
├── fs           File and directory operations, path handling, watchers
├── os           Env vars, process management, signals, system info
├── time         Clocks, timers, duration, timezone, formatting
│
├── async        Event loop, spawn, futures, channels, sync primitives
│                Supervisor, TaskGroup, CircuitBreaker, retry, timeout
│
├── net          TCP, UDP, Unix sockets, DNS resolver
├── crypto       SHA, AES, RSA, ECDSA, Ed25519, TLS 1.3, X.509
├── http         HTTP/1.1 + HTTP/2 client and server, WebSocket, router
│
├── json         JSON parse/serialize, streaming, JSON Path
├── xml          XML SAX/DOM parser, serialization, XPath
│
├── db           Common database interface (Connection, Pool, Transaction)
│   ├── mysql    MySQL/MariaDB (wire protocol, no libmysqlclient)
│   ├── mssql    SQL Server (TDS protocol, no ODBC)
│   └── mongodb  MongoDB (wire protocol, no libmongoc)
│
├── log          Structured logging (JSON, logfmt, human), levels, sampling
├── test         Test framework, assertions, benchmarks, fuzzing
└── encoding     Base64, hex, URL encoding, UTF conversions
```

## Design Principles

### 1. Zero External Dependencies
Every stdlib package is implemented from scratch in Limceron. No C bindings, no system
libraries (except the OS kernel interface). This means:
- Database drivers implement the wire protocol directly
- TLS is implemented in Limceron (no OpenSSL/BoringSSL)
- HTTP is implemented in Limceron (no libcurl)
- Cross-compilation works without installing target system libraries

### 2. Consistent Error Handling
Every function that can fail returns `Result<T>`. Errors carry:
- A human-readable message
- An error code (for programmatic matching)
- An optional cause chain
- Source location where the error originated

```limceron
interface Error {
    fn message(&self) -> string
    fn code(&self) -> string        // e.g., "ECONNREFUSED", "ENOENT"
    fn cause(&self) -> Option<&Error>
    fn location(&self) -> Location  // file, line, column
}
```

### 3. Production-Grade Defaults
- Connection pools have health checks enabled by default
- HTTP client follows redirects (up to 10) by default
- TLS certificate validation is on by default (opt-out, not opt-in)
- Logging includes timestamp and caller location by default
- Timeouts have sensible defaults (30s for HTTP, 10s for DNS)

### 4. Fault Tolerance Built In
The `async` package includes production patterns:
- `retry` — configurable retry with exponential backoff
- `timeout` — hard timeout for any operation
- `CircuitBreaker` — prevent cascade failures
- `Bulkhead` — limit concurrency to a resource
- `RateLimiter` — token bucket rate limiting
- `Supervisor` — restart failed tasks automatically
- `Context` — cancellation propagation (like Go's context)

### 5. Cross-Platform Abstraction
Platform-specific code is isolated behind consistent interfaces:
- `fs.Path` normalizes paths across Windows/Unix
- `net` handles socket differences (Winsock vs POSIX)
- `os` abstracts signals, process management
- `async` uses the best event loop per platform (epoll/kqueue/IOCP)
- `comptime if` for platform-specific implementations

## Import Convention

```limceron
use io                  // standard library
use json
use db.mysql
use http

use github.com/user/pkg // third-party (future package registry)

use myapp.config        // local project module
```

Standard library packages are always a single word or `parent.child` — never a URL.
Third-party packages use the full repository path.
