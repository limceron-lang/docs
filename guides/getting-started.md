# Getting Started with Limceron

## Installation

### From Source (Bootstrap)
```bash
git clone https://github.com/limceron-lang/limceron
cd limceron
make bootstrap    # Builds stage0 (C) → stage1 (Limceron subset) → stage2 (full Limceron)
make install      # Installs `limceron` to /usr/local/bin
```

### Verify Installation
```bash
limceron version
# Limceron 0.1.0 (linux-x86_64)
```

## Your First Program

### 1. Create a Project
```bash
limceron init hello-world
cd hello-world
```

This creates:
```
hello-world/
├── limceron.pkg          # package manifest
├── main.lceron           # entry point
└── main_test.lceron      # test file
```

### 2. The Entry Point

`main.lceron`:
```limceron
mod main

fn main() -> Result<void> {
    print("Hello, World!")
    Ok(())
}
```

Every Limceron program starts with a `main` function that returns `Result<void>`.
This means every program handles errors from the start — no unhandled panics in production.

### 3. Build and Run
```bash
limceron run              # compile and run
limceron build            # compile to binary
./hello-world         # run the binary
```

### 4. Run Tests
```bash
limceron test             # run all tests
limceron test ./...       # run tests recursively
```

---

## Language Basics

### Variables
```limceron
let name = "Limceron"              // immutable (default)
let mut counter = 0              // mutable
counter += 1                     // OK
// name = "other"                // COMPILE ERROR: immutable

// Type annotation (usually optional)
let port: u16 = 8080
```

### Functions
```limceron
fn greet(name: string) -> string {
    "Hello, {name}!"             // last expression is the return value
}

// Multiple return values via tuples
fn divide(a: f64, b: f64) -> Result<f64> {
    if b == 0.0 {
        return Err("division by zero")
    }
    Ok(a / b)
}
```

### Structs
```limceron
struct User {
    name: string
    email: string
    age: u8
}

impl User {
    fn new(name: string, email: string, age: u8) -> User {
        User { name, email, age }
    }

    fn is_adult(&self) -> bool {
        self.age >= 18
    }

    fn birthday(&mut self) {
        self.age += 1
    }
}

// Usage
let mut user = User.new("Alice", "alice@example.com", 30)
user.birthday()
print(user.is_adult())  // true
```

### Enums
```limceron
enum Color {
    Red
    Green
    Blue
    Custom(r: u8, g: u8, b: u8)
}

fn describe(c: Color) -> string {
    match c {
        Color.Red -> "red"
        Color.Green -> "green"
        Color.Blue -> "blue"
        Color.Custom(r, g, b) -> "rgb({r}, {g}, {b})"
    }
}
```

### Error Handling
```limceron
// Functions that can fail return Result<T>
fn read_config(path: string) -> Result<Config> {
    let content = fs.read_file(path)?       // ? propagates errors
    let config = json.parse<Config>(content)?
    Ok(config)
}

// Handle errors explicitly
match read_config("app.json") {
    Ok(config) -> start_server(config)
    Err(e) -> {
        log.error("failed to read config: {e.message()}")
        os.exit(1)
    }
}

// Or provide defaults
let config = read_config("app.json").unwrap_or(Config.default())
```

### Collections
```limceron
// Vector (growable array)
let mut items = vec![1, 2, 3]
items.push(4)

// HashMap
let mut scores = #{
    "alice": 100,
    "bob": 85,
}
scores["charlie"] = 92

// Iteration
for name, score in scores {
    print("{name}: {score}")
}

// Functional style
let high_scores = scores
    .iter()
    .filter(|(_, s)| s > 90)
    .map(|(name, _)| name)
    .collect<Vec<string>>()
```

### Concurrency
```limceron
// Spawn a task (lightweight green thread)
let handle = spawn {
    expensive_computation()
}
let result = await handle

// Channels
let ch = chan<string>(buffer: 10)

spawn {
    ch.send("hello from task")
}

let msg = ch.recv()?
print(msg)

// Parallel work with TaskGroup
let group = TaskGroup.new()
for url in urls {
    group.spawn(|| http.get(url))
}
let responses = group.wait()?
```

---

## Common Patterns

### HTTP Server
```limceron
use http

fn main() -> Result<void> {
    let mut server = http.Server.new()

    server.get("/", |req| {
        Ok(http.Response.text("Hello, Limceron!"))
    })

    server.get("/users/{id}", |req| {
        let id = req.param("id")?
        let user = db.find_user(id)?
        Ok(http.Response.json(user))
    })

    server.post("/users", |req| {
        let body = req.json<CreateUser>()?
        let user = db.create_user(body)?
        Ok(http.Response.json(user).status(201))
    })

    log.info("server starting on :8080")
    server.listen(":8080")?
    Ok(())
}
```

### Database Query
```limceron
use db.mysql

fn main() -> Result<void> {
    let pool = mysql.Pool.new(
        host: "localhost",
        port: 3306,
        user: "root",
        password: os.env("DB_PASSWORD")?,
        database: "myapp",
        max_connections: 10,
    )?

    defer pool.close()

    // Query with automatic struct mapping
    let users = pool.query<User>(
        "SELECT id, name, email FROM users WHERE active = ?",
        true
    )?

    for user in users {
        print("{user.name}: {user.email}")
    }

    // Transaction
    let tx = pool.begin()?
    tx.exec("UPDATE accounts SET balance = balance - ? WHERE id = ?", 100, from_id)?
    tx.exec("UPDATE accounts SET balance = balance + ? WHERE id = ?", 100, to_id)?
    tx.commit()?

    Ok(())
}
```

### JSON Processing
```limceron
use json

#[json(rename_all = "camelCase")]
struct ApiResponse {
    user_name: string
    is_active: bool
    #[json(name = "scores")]
    test_scores: Vec<int>
}

fn main() -> Result<void> {
    // Parse
    let data = `{"userName": "alice", "isActive": true, "scores": [95, 87, 92]}`
    let resp = json.parse<ApiResponse>(data)?

    // Serialize
    let output = json.stringify(resp, indent: 2)?
    print(output)

    Ok(())
}
```

### File Operations
```limceron
use fs
use io

fn process_log(path: string) -> Result<int> {
    let file = fs.open(path)?
    defer file.close()

    let reader = io.BufferedReader.new(file)
    let mut error_count = 0

    for line in reader.lines() {
        let line = line?
        if line.contains("ERROR") {
            error_count += 1
        }
    }

    Ok(error_count)
}
```

---

## Project Structure

### Small Project
```
my-app/
├── limceron.pkg
├── main.lceron
└── main_test.lceron
```

### Medium Project
```
my-service/
├── limceron.pkg
├── main.lceron
├── config/
│   ├── config.lceron
│   └── config_test.lceron
├── handler/
│   ├── users.lceron
│   ├── posts.lceron
│   └── handler_test.lceron
├── model/
│   ├── user.lceron
│   └── post.lceron
└── db/
    ├── connection.lceron
    └── queries.lceron
```

### Large Project
```
my-platform/
├── limceron.pkg
├── cmd/
│   ├── server/main.lceron       # server binary
│   ├── worker/main.lceron       # background worker binary
│   └── cli/main.lceron          # CLI tool binary
├── pkg/                     # shared library code
│   ├── auth/
│   ├── billing/
│   └── notification/
├── internal/                # private to this project
│   ├── metrics/
│   └── middleware/
└── tests/
    └── integration/
```

---

## Agent Memory & Knowledge Base

### Persistent Memory

Add `## memory` to any agent to enable SQLite-backed persistent memory:

```markdown
# agent Assistant
> You are a helpful assistant.
## capabilities
- llm.complete
## model
llama3
## memory
true
## budget
- max_tokens: 10000
- max_cost: 1
```

Build and run:
```bash
limceron build assistant.lceron.md -o assistant
./assistant "Hello"          # response stored in lcn_memory.db
./assistant "What did I say?" # agent recalls previous interactions
```

Files created:
- `lcn_memory.db` — SQLite database with entries and sessions
- `lcn_memory.jsonl` — append-only audit trail

Environment variables:
- `LCN_MEMORY_DB` — custom database path (default: `lcn_memory.db`)

### Knowledge Base (RAG)

Point an agent at a directory of documents for automatic RAG:

```markdown
# agent DocBot
> You answer questions based on documentation.
## capabilities
- llm.complete
## model
claude-haiku
## knowledge
- path: ./docs
- chunk_size: 500
- overlap: 50
## budget
- max_tokens: 50000
- max_cost: 5
```

```bash
limceron build docbot.lceron.md -o docbot
./docbot "How do I configure the API?"
# First run: ingests ./docs, chunks files, indexes with FTS5
# Subsequent runs: instant startup from cached lcn_kb.db
```

Knowledge config:
- `path` — directory containing documents (required)
- `chunk_size` — characters per chunk (default: 500)
- `overlap` — overlap between chunks (default: 50)

Supported formats: `.txt`, `.md`, `.json`, `.csv`, `.yaml`, `.xml`, `.html`, `.rst`, `.toml`, `.ini`, `.cfg`, `.conf`, `.log`

Environment variables:
- `LCN_KB_DB` — custom KB database path (default: `lcn_kb.db`)

### Dashboard API

Enable the dashboard with `LCN_DASHBOARD=1`:
```bash
LCN_DASHBOARD=1 ./docbot "Hello" &
curl http://localhost:9090/api/memory?agent=DocBot
curl http://localhost:9090/api/memory/sessions
curl http://localhost:9090/api/kb/status
curl "http://localhost:9090/api/kb/search?q=configure+API"
```

## Build Commands Reference

| Command              | Description                                    |
|----------------------|------------------------------------------------|
| `limceron build`         | Compile the current package                    |
| `limceron run`           | Compile and run                                |
| `limceron test`          | Run tests in current package                   |
| `limceron test ./...`    | Run all tests recursively                      |
| `limceron fmt`           | Format source code                             |
| `limceron fmt --check`   | Check formatting without modifying             |
| `limceron lint`          | Run linter                                     |
| `limceron doc`           | Generate documentation                         |
| `limceron deps add x`    | Add dependency                                 |
| `limceron deps update`   | Update dependencies                            |
| `limceron deps vendor`   | Vendor all dependencies                        |
| `limceron build --target=T` | Cross-compile for target T                  |
| `limceron targets`       | List available compilation targets             |
| `limceron version`       | Show compiler version                          |
| `limceron vet`           | Static analysis                                |
| `limceron bench`         | Run benchmarks                                 |
| `limceron fuzz`          | Run fuzz tests                                 |
