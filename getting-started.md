# Getting Started with Limceron

Build and run your first AI agent in 5 minutes.

## Prerequisites

- **gcc** or **clang** (C99-compatible)
- **make**
- Optional: `libmysqlclient-dev` (for `use driver("mysql")`)
- Optional: an LLM endpoint (Ollama, vLLM, or any OpenAI-compatible API)

## Build

```bash
git clone <repo-url> limceron
cd limceron
make stage0        # builds the bootstrap compiler
make test          # 238 tests, 960 assertions -- all must pass
```

The compiler binary is at `build/limceron-stage0`. You can copy it to your PATH:

```bash
cp build/limceron-stage0 /usr/local/bin/lcn
```

## Your First Agent

### Option A: Full Syntax (.lceron)

Create `hello.lceron`:

```limceron
capability llm {
    complete
}

budget HelloBudget {
    max_tokens: 5000
    max_cost: 0.50
}

agent Hello {
    capabilities: [llm.complete]
    model: "llama3"
    budget: HelloBudget
    prompt: "You are a helpful assistant."

    fn run(question: string) -> Result {
        let answer = ask(question)
        answer
    }
}

fn main() -> Result {
    let agent = Hello
    let result = agent.run("What is Limceron?")
    result
}
```

Build and run:

```bash
lcn build hello.lceron -o hello
./hello
```

### Option B: Markdown-as-Source (.lceron.md)

Create `hello.lceron.md`:

````markdown
# agent Hello
> You are a helpful assistant.
## capabilities
- llm.complete
## model
llama3
## budget
- max_tokens: 5000
- max_cost: 0.50
````

Build and run:

```bash
lcn build hello.lceron.md -o hello
./hello "What is Limceron?"
```

Markdown-as-source is first-class. The compiler extracts agent definitions directly from Markdown headings and lists. No code blocks needed.

## Key Concepts

### Agents

An agent is a unit of autonomous behavior: it has a model, capabilities, a budget, and methods that call `ask()` to query the LLM.

### Capabilities

Capabilities declare what an agent is allowed to do. The type checker enforces that every tool call has matching capability grants. Capabilities also support concrete access control rules:

```limceron
capability web_access {
    allow endpoint "api.example.com:443" {
        method: [GET, POST]
        path: "/v1/**"
    }
    deny private_ranges
    default: deny
}
```

### ask() and Typed LLM Output

`ask()` returns a typed `LLMOutput` value. Use `match` to handle every case:

```limceron
let result = ask(prompt)
match result {
    Ok(text) -> process(text)
    Error(e) -> log("LLM error: {e}")
}
```

When the return type is an enum, the compiler generates a constrained prompt so the LLM must respond with a valid variant.

### Access Control Policies

Network endpoints, filesystem paths, and binary execution are all governed by capability rules. The type checker flags violations at compile time; the runtime enforces them at execution time.

### Budgets and Entropy

Budgets cap token usage, cost, and duration. Agents using tools must declare a budget (enforced by the type checker). Optional `entropy_budget` tracks decision randomness:

```limceron
agent Cautious {
    budget: ConservativeBudget
    entropy_budget: {
        max_entropy_per_decision: 1.5
        max_cumulative: 10.0
    }
}
```

### Taint Tracking

User input is tainted at compile time. The type checker prevents tainted strings from flowing into system prompts without sanitization:

```limceron
taint user_input

fn handle(msg: string@user_input) -> Result {
    // msg cannot be passed directly to ask() -- must sanitize first
    let clean = sanitize(msg)
    ask(clean)
}
```

## CLI Commands

| Command | Description |
|---------|-------------|
| `lcn build <file> [-o output]` | Compile a .lceron or .lceron.md to a native binary |
| `lcn run <file>` | Compile and run in one step |
| `lcn emit <file>` | Show generated C code (useful for debugging) |
| `lcn parse <file>` | Print the AST |
| `lcn lex <file>` | Print the token stream |
| `lcn audit <file>` | Security audit: capabilities, budgets, access rules |
| `lcn init <name>` | Create a new project skeleton |

### LLM Endpoint Configuration

Set the LLM endpoint via environment variable:

```bash
export LCN_LLM_ENDPOINT="http://localhost:11434"   # Ollama
# or
export LCN_LLM_ENDPOINT="http://gpu-server:8000"   # vLLM
```

The compiler checks `LCN_LLM_ENDPOINT`, then `OLLAMA_HOST`, then falls back to `http://localhost:11434`.

## Examples

The `examples/language/` directory contains 23+ working examples:

| File | Topic |
|------|-------|
| `01_hello_agent.lceron` | Minimal agent |
| `02_budget_limits.lceron` | Token and cost budgets |
| `03_capabilities.lceron` | Capability declarations and verification |
| `07_guards.lceron` | Guard functions (human-in-the-loop, dry-run) |
| `08_taint_types.lceron` | Taint tracking for prompt injection prevention |
| `15_mcp_a2a.lceron` | MCP and A2A protocol integration |
| `20_medical_categorizer.lceron` | Production pattern: MySQL driver, access control, guards |
| `22_taint_demo.lceron` | Compile-time taint safety demo |
| `23_access_control.lceron` | Endpoint, binary, and path access policies |

Run any example:

```bash
lcn run examples/language/01_hello_agent.lceron
```

Or build and inspect:

```bash
lcn emit examples/language/20_medical_categorizer.lceron   # see generated C
lcn audit examples/language/23_access_control.lceron        # security report
```

## Multi-file Projects

Limceron supports multi-file projects with `mod` and `use`:

```
project/
  main.lceron        # mod main; use helpers
  helpers.lceron     # mod helpers; pub fn format(...)
```

```bash
lcn build project/main.lceron -o myapp
```

Only `pub` declarations are visible across modules.

## Next Steps

- **Specification**: `spec/specification.md` -- full language reference
- **Grammar**: `spec/grammar.ebnf` -- formal EBNF grammar
- **Production example**: `examples/language/20_medical_categorizer.lceron`
- **Existing guide**: `docs/guides/getting-started.md` -- language basics and patterns
- **Architecture**: `docs/architecture/` -- compiler internals
