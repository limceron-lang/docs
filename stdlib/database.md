# Limceron Database Connectors

## Common Interface

All database connectors implement the same core interfaces. You can write code that
works with any database by programming against these interfaces.

```limceron
// Core interfaces (defined in db module)
interface Connection {
    fn query<T>(sql: string, params: ...any) -> Result<Rows<T>>
    fn exec(sql: string, params: ...any) -> Result<ExecResult>
    fn begin() -> Result<Transaction>
    fn close() -> Result<void>
    fn ping() -> Result<void>
}

interface Transaction {
    fn query<T>(sql: string, params: ...any) -> Result<Rows<T>>
    fn exec(sql: string, params: ...any) -> Result<ExecResult>
    fn commit() -> Result<void>
    fn rollback() -> Result<void>
}

interface Pool {
    fn acquire() -> Result<Connection>
    fn release(conn: Connection)
    fn stats() -> PoolStats
    fn close() -> Result<void>
}

struct PoolConfig {
    max_connections: int = 10
    min_connections: int = 1
    max_idle_time: Duration = 5.minutes()
    max_lifetime: Duration = 1.hours()
    health_check_interval: Duration = 30.seconds()
    connect_timeout: Duration = 5.seconds()
    query_timeout: Duration = 30.seconds()
}

struct ExecResult {
    rows_affected: i64
    last_insert_id: i64?
}

struct PoolStats {
    total: int
    idle: int
    in_use: int
    wait_count: i64
    wait_duration: Duration
}
```

## Row Scanning

Limceron automatically maps database rows to structs using field names:

```limceron
struct User {
    id: i64
    name: string
    email: string
    created_at: DateTime
    is_active: bool
}

// Automatic mapping: column names match struct field names
let users = pool.query<User>("SELECT id, name, email, created_at, is_active FROM users")?

for user in users {
    print("{user.name}: {user.email}")
}

// Custom column mapping with attributes
#[db(table = "users")]
struct UserDTO {
    id: i64
    #[db(column = "full_name")]
    name: string
    email: string
    #[db(skip)]
    computed_field: string = ""
}
```

## Parameterized Queries

All connectors use `?` as the parameter placeholder (normalized across databases):

```limceron
// Positional parameters
let users = pool.query<User>(
    "SELECT * FROM users WHERE age > ? AND city = ?",
    18, "Madrid"
)?

// The connector handles type conversion automatically:
// string → VARCHAR/TEXT
// int/i64 → BIGINT
// f64 → DOUBLE
// bool → BOOLEAN/BIT
// DateTime → DATETIME/TIMESTAMP
// Option<T> → nullable
// Vec<u8> → BLOB/BINARY
```

---

## MySQL / MariaDB

### Connection
```limceron
use db.mysql

let pool = mysql.Pool.new(
    host: "localhost",
    port: 3306,
    user: "root",
    password: os.env("MYSQL_PASSWORD")?,
    database: "myapp",
    pool: PoolConfig {
        max_connections: 20,
        connect_timeout: 3.seconds(),
    },
    tls: mysql.TlsConfig.Preferred,  // Preferred | Required | Disabled
)?
defer pool.close()

// Or with DSN string
let pool = mysql.Pool.from_dsn(
    "mysql://root:pass@localhost:3306/myapp?tls=preferred"
)?
```

### Queries
```limceron
// Simple query
let users = pool.query<User>("SELECT * FROM users WHERE active = ?", true)?

// Single row
let user = pool.query_one<User>("SELECT * FROM users WHERE id = ?", 42)?
// Returns Option<User> — None if no row found

// Execute (INSERT, UPDATE, DELETE)
let result = pool.exec(
    "INSERT INTO users (name, email) VALUES (?, ?)",
    "Alice", "alice@example.com"
)?
print("Inserted ID: {result.last_insert_id}")

// Batch insert
let users = vec![
    ("Alice", "alice@example.com"),
    ("Bob", "bob@example.com"),
    ("Charlie", "charlie@example.com"),
]
let result = pool.exec_batch(
    "INSERT INTO users (name, email) VALUES (?, ?)",
    users
)?
print("Inserted {result.rows_affected} rows")
```

### Transactions
```limceron
let tx = pool.begin()?

// If anything fails, the transaction is automatically rolled back
// when tx goes out of scope (via defer/Drop)
tx.exec("UPDATE accounts SET balance = balance - ? WHERE id = ?", amount, from_id)?
tx.exec("UPDATE accounts SET balance = balance + ? WHERE id = ?", amount, to_id)?
tx.exec(
    "INSERT INTO transfers (from_id, to_id, amount) VALUES (?, ?, ?)",
    from_id, to_id, amount
)?

tx.commit()?  // explicit commit required
```

### Prepared Statements
```limceron
let stmt = pool.prepare("SELECT * FROM users WHERE department = ? AND role = ?")?
defer stmt.close()

let engineers = stmt.query<User>("engineering", "senior")?
let managers = stmt.query<User>("engineering", "manager")?
```

---

## Microsoft SQL Server

### Connection
```limceron
use db.mssql

let pool = mssql.Pool.new(
    host: "localhost",
    port: 1433,
    user: "sa",
    password: os.env("MSSQL_PASSWORD")?,
    database: "myapp",
    pool: PoolConfig {
        max_connections: 20,
    },
    tls: mssql.TlsConfig.Required,
    // Windows auth (Windows only):
    // auth: mssql.Auth.Windows,
)?
defer pool.close()

// DSN format
let pool = mssql.Pool.from_dsn(
    "mssql://sa:pass@localhost:1433/myapp?encrypt=true"
)?
```

### SQL Server-Specific Features
```limceron
// OUTPUT clause
let result = pool.query<InsertedRow>(
    "INSERT INTO users (name, email) OUTPUT INSERTED.id, INSERTED.created_at VALUES (?, ?)",
    "Alice", "alice@example.com"
)?

// Stored procedures
let result = pool.exec("EXEC sp_update_user @id = ?, @name = ?", 42, "Alice")?

// Bulk copy (high-performance bulk insert)
let rows = vec![
    UserRow { name: "Alice", email: "alice@example.com" },
    UserRow { name: "Bob", email: "bob@example.com" },
    // ... thousands of rows
]
let copied = pool.bulk_copy("users", rows)?
print("Copied {copied} rows")

// Multiple result sets
let results = pool.query_multi("SELECT * FROM users; SELECT * FROM orders")?
let users = results.next<User>()?
let orders = results.next<Order>()?
```

---

## MongoDB

### Connection
```limceron
use db.mongodb

let client = mongodb.Client.new(
    uri: "mongodb://localhost:27017",
    database: "myapp",
    pool: PoolConfig {
        max_connections: 20,
    },
    tls: mongodb.TlsConfig.Disabled,
    // Replica set:
    // replica_set: "rs0",
    // read_preference: mongodb.ReadPreference.SecondaryPreferred,
)?
defer client.close()

let db = client.database("myapp")
let users = db.collection<User>("users")
```

### CRUD Operations
```limceron
// Insert
let user = User { name: "Alice", email: "alice@example.com", age: 30 }
let result = users.insert_one(user)?
print("Inserted: {result.inserted_id}")

// Insert many
let new_users = vec![user1, user2, user3]
let result = users.insert_many(new_users)?

// Find one
let user = users.find_one(#{ "email": "alice@example.com" })?

// Find many with filter
let active_users = users.find(#{
    "is_active": true,
    "age": #{ "$gte": 18 },
})?
.sort(#{ "name": 1 })
.limit(100)
.exec()?

for user in active_users {
    print(user.name)
}

// Update
let result = users.update_one(
    #{ "_id": user_id },
    #{ "$set": #{ "name": "Alice Smith" } },
)?
print("Modified: {result.modified_count}")

// Delete
let result = users.delete_many(#{ "is_active": false })?
print("Deleted: {result.deleted_count}")
```

### Aggregation Pipeline
```limceron
let pipeline = vec![
    #{ "$match": #{ "status": "active" } },
    #{ "$group": #{
        "_id": "$department",
        "count": #{ "$sum": 1 },
        "avg_salary": #{ "$avg": "$salary" },
    }},
    #{ "$sort": #{ "count": -1 } },
]

let results = users.aggregate<DeptStats>(pipeline)?
for dept in results {
    print("{dept._id}: {dept.count} employees, avg salary: {dept.avg_salary}")
}
```

### Change Streams
```limceron
let stream = users.watch(
    pipeline: vec![
        #{ "$match": #{ "operationType": "insert" } },
    ],
)?

for change in stream {
    let change = change?
    match change.operation {
        "insert" -> print("New user: {change.document}")
        "update" -> print("Updated: {change.document_key}")
        "delete" -> print("Deleted: {change.document_key}")
        _ -> {}
    }
}
```

### Indexes
```limceron
// Create index
users.create_index(
    keys: #{ "email": 1 },
    options: mongodb.IndexOptions {
        unique: true,
        name: "idx_email",
    },
)?

// Compound index
users.create_index(
    keys: #{ "department": 1, "role": 1 },
    options: mongodb.IndexOptions {
        name: "idx_dept_role",
    },
)?

// Text index
users.create_index(
    keys: #{ "name": "text", "bio": "text" },
    options: mongodb.IndexOptions {
        name: "idx_text_search",
    },
)?
```

---

## Connection Pool Best Practices

```limceron
// Production pool configuration
let pool = mysql.Pool.new(
    // ...connection details...
    pool: PoolConfig {
        // Size: match your workload, not your max
        max_connections: 20,
        min_connections: 5,

        // Recycle connections to prevent stale state
        max_lifetime: 30.minutes(),
        max_idle_time: 5.minutes(),

        // Health checks catch dead connections
        health_check_interval: 30.seconds(),

        // Don't wait forever for a connection
        connect_timeout: 3.seconds(),

        // Don't let queries run forever
        query_timeout: 30.seconds(),
    },
)?

// Monitor pool health
spawn {
    loop {
        let stats = pool.stats()
        log.info("pool stats",
            total: stats.total,
            idle: stats.idle,
            in_use: stats.in_use,
            waiting: stats.wait_count,
        )
        time.sleep(60.seconds())
    }
}
```

## Error Handling

```limceron
use db
use db.mysql

fn get_user(pool: &mysql.Pool, id: i64) -> Result<User> {
    let user = pool.query_one<User>(
        "SELECT * FROM users WHERE id = ?", id
    )?;

    match user {
        Some(u) -> Ok(u)
        None -> Err(AppError.NotFound("user", id))
    }
}

// Database errors carry structured information
fn handle_db_error(err: &db.Error) {
    match err {
        db.Error.ConnectionFailed(e) -> {
            log.error("database unreachable: {e.message()}")
            // trigger alert, use circuit breaker
        }
        db.Error.QueryFailed(e) -> {
            log.error("query error: {e.message()}, code: {e.code()}")
        }
        db.Error.Timeout -> {
            log.warn("query timed out")
        }
        db.Error.PoolExhausted -> {
            log.error("connection pool exhausted")
            // consider increasing pool size or adding backpressure
        }
    }
}
```
