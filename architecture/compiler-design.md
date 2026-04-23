# Limceron Compiler Architecture

## Overview

The Limceron compiler is a single-pass, multi-phase compiler that transforms `.lceron` source files
into native executables. It uses no external dependencies beyond libc (for the stage 0
bootstrap) and is designed for fast compilation with excellent error messages.

```
Source (.lceron)
    |
    v
+----------+    +----------+    +----------+    +----------+    +----------+
|  Lexer   |--->|  Parser  |--->|  Type    |--->|   IR     |--->| Codegen  |
|          |    |          |    |  Checker |    |  (SSA)   |    |          |
+----------+    +----------+    +----------+    +----------+    +----------+
                                                                     |
                                                                     v
                                                              +----------+
                                                              |  Linker  |
                                                              +----------+
                                                                     |
                                                                     v
                                                              Native Binary
                                                         (ELF / Mach-O / PE)
```

## Phase 1: Lexical Analysis (Lexer)

**Input:** UTF-8 source bytes
**Output:** Token stream

- Hand-written lexer, no regex or generators
- Single-pass, character-by-character
- UTF-8 aware (identifiers support Unicode letters)
- Tracks line, column, and byte offset for every token
- Automatic semicolon insertion (Go-style rules)
- Produces `Token` values:

```
Token {
    kind: TokenKind      // Ident, IntLit, StringLit, Plus, LParen, etc.
    span: Span           // file_id, start_offset, end_offset
    value: TokenValue    // interned string, parsed number, etc.
}
```

**Memory:** Tokens reference the source buffer directly (zero-copy for identifiers/strings).
Tokens are allocated in a per-file arena.

## Phase 2: Parsing

**Input:** Token stream
**Output:** Abstract Syntax Tree (AST)

- Recursive descent parser for statements and declarations
- Pratt parser (precedence climbing) for expressions
- Error recovery: on syntax error, skip to next synchronization point
  (`;`, `}`, `fn`, `struct`, etc.) and continue parsing
- Reports all parse errors, not just the first one
- Produces a typed AST with `Span` on every node for error reporting

```
AstNode {
    kind: AstKind        // FnDecl, StructDecl, BinaryExpr, CallExpr, etc.
    span: Span           // source location
    children: [AstNode]  // child nodes
    data: AstData        // kind-specific data
}
```

**Memory:** AST nodes allocated in a per-module arena. The token arena can be freed after
parsing completes.

## Phase 3: Name Resolution and Type Checking

**Input:** AST per module
**Output:** Typed AST (TAST) with resolved symbols

### 3a: Name Resolution
- Build symbol table for each module
- Resolve imports (`use` declarations)
- Resolve all identifier references to their declarations
- Detect naming conflicts, unused imports, shadowing

### 3b: Type Inference and Checking
- Hindley-Milner style inference with extensions for ownership
- Bidirectional type checking (push expected types down, pull inferred types up)
- Monomorphization of generics (like Rust, not like Java erasure)
- Region inference for references (simplified lifetime analysis)
- Exhaustiveness checking for `match` expressions
- Ownership and borrowing analysis:
  - Track move/copy semantics
  - Verify borrow rules (one `&mut` XOR many `&`)
  - Detect use-after-move
  - Verify `unsafe` boundaries

### 3c: Trait/Interface Resolution
- Resolve trait implementations (explicit `impl`)
- Resolve interface satisfaction (structural matching)
- Detect ambiguous implementations
- Generate vtables for dynamic dispatch

**Memory:** TAST allocated in a new arena. AST arena freed after type checking.

## Phase 4: IR Generation (SSA Form)

**Input:** Typed AST
**Output:** Limceron IR (LIR) in SSA form

The IR is a low-level, typed, SSA (Static Single Assignment) representation:

```
LIR Instruction {
    opcode: OpCode       // Add, Load, Store, Call, Branch, Phi, etc.
    type: LIRType        // i32, i64, ptr, f64, void, etc.
    operands: [Value]    // virtual registers or constants
    block: BlockId       // containing basic block
}
```

### IR Optimization Passes (ordered)
1. **Dead code elimination** — remove unreachable blocks and unused values
2. **Constant folding** — evaluate constant expressions at compile time
3. **Inlining** — inline small functions and always-inline functions
4. **Escape analysis** — promote heap allocations to stack when possible
5. **Common subexpression elimination** — avoid redundant computation
6. **Loop-invariant code motion** — hoist invariant code out of loops
7. **Strength reduction** — replace expensive ops (multiply -> shift)
8. **Devirtualization** — convert trait calls to direct calls when type is known
9. **Tail call optimization** — convert tail calls to jumps
10. **Dead store elimination** — remove stores that are never read

**Memory:** IR allocated in its own arena. TAST arena freed after IR generation.

## Phase 5: Code Generation

**Input:** Optimized LIR
**Output:** Machine code in `.nobj` format

### Register Allocation
- Linear scan register allocator (fast, good enough for most code)
- Optional graph coloring for hot functions (when PGO data is available)
- Spill to stack with intelligent spill cost analysis

### Instruction Selection
- Pattern-matching based instruction selection
- Platform-specific patterns for each target:

```
Target backends:
+-- x86_64     (Linux ELF, macOS Mach-O, Windows PE)
+-- aarch64    (Linux ELF, macOS Mach-O, Windows PE)
+-- riscv64    (Linux ELF)
```

### Machine Code Emission
- Direct binary emission (no assembler dependency)
- Relocation table for linker
- Debug info generation (DWARF for Unix, CodeView for Windows)
- Stack maps for precise GC of `Rc`/`Arc` (future)

## Phase 6: Linking

**Input:** `.nobj` files (all modules)
**Output:** Native executable

The linker is fully integrated — no external `ld`, `link.exe`, or `lld` needed.

### Link Process
1. Read all `.nobj` files
2. Resolve cross-module references
3. Perform link-time optimizations (optional):
   - Cross-module inlining
   - Whole-program dead code elimination
   - Identical code folding
4. Layout sections (text, data, rodata, bss)
5. Apply relocations
6. Write output in platform format:
   - **Linux:** ELF64
   - **macOS:** Mach-O 64
   - **Windows:** PE/COFF (PE32+)
7. Static linking by default (no shared library dependencies except libc/system libs)

## Incremental Compilation

### Change Detection
- Content hash of each source file
- Dependency graph between modules
- If module A imports module B, and B's public interface changes, A must be recompiled
- If only B's implementation changes (not its public interface), A is NOT recompiled

### Cache Strategy
```
.lceron-cache/
+-- module_hashes.json     # content hash of each module
+-- dep_graph.json         # module dependency graph
+-- objects/               # cached .nobj files
|   +-- math.vector.nobj
|   +-- http.server.nobj
|   +-- ...
+-- metadata/              # cached type information per module
    +-- math.vector.meta
    +-- ...
```

## Parallel Compilation

- Modules without dependencies on each other can be compiled in parallel
- The compiler builds a dependency DAG and schedules compilation in topological order
- Within each "level" of the DAG, modules compile concurrently
- Thread pool size = number of CPU cores (configurable)

```
Level 0: [core]                          (1 module, sequential)
Level 1: [io, fmt, math, collections]    (4 modules, parallel)
Level 2: [fs, net, crypto]              (3 modules, parallel)
Level 3: [http, json, xml]              (3 modules, parallel)
Level 4: [db.mysql, db.mssql, db.mongodb] (3 modules, parallel)
Level 5: [app.main]                      (1 module, sequential)
```

## Error Reporting

Errors are reported with:
1. **Location:** File, line, column, with source snippet
2. **Message:** What went wrong (plain English)
3. **Explanation:** Why it's wrong
4. **Suggestion:** How to fix it (with suggested code when possible)

```
error[E0312]: cannot borrow `items` as mutable because it is also borrowed as immutable

  --> src/main.lceron:15:5
   |
13 |     let first = &items[0]
   |                  ------ immutable borrow occurs here
14 |
15 |     items.push(42)
   |     ^^^^^ mutable borrow occurs here
16 |
17 |     print(first)
   |           ----- immutable borrow later used here
   |
   = help: consider cloning the value: `let first = items[0].clone()`
```

## Cross-Compilation

Cross-compilation is built into the compiler, not an external tool:

```bash
# Compile for a different target
limceron build --target=linux-arm64
limceron build --target=windows-x86_64
limceron build --target=linux-riscv64

# List available targets
limceron targets
```

The compiler carries codegen backends for all supported targets in a single binary.
No separate toolchain installation needed.
