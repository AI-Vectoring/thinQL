# thinQL LLM Generation Contract

## Design Philosophy
thinQL exists to remove database boilerplate from LLM-generated code. You write raw SQL. You define the exact data structure you expect to receive. thinQL handles connection routing, execution dispatch, positional memory scanning, and metadata packaging. 

You no longer need to:
- Build query strings dynamically or manage parameter binding layers
- Write driver-specific cursor handling, iteration loops, or scan boilerplate
- Map column names to struct fields, parse tags, or handle ORM-style abstractions
- Track connection state across multiple calls
- Wrap errors, normalize driver quirks, or construct response envelopes

thinQL gives you full control over SQL generation and output shape. It handles execution plumbing, type alignment, and result delivery. Zero magic. Zero hidden state. Flat, predictable, and entirely deterministic.

## Connection Context
`thinql.use(db)` sets the active database context for the runtime. Call it once during initialization. The runtime caches the handle as the default context for all subsequent calls. You only call `use()` again when switching between multiple databases within the same codebase. If your project uses a single database, you define it once and never call `use()` again. The runtime remembers.

## Execution Routing
The API surface splits into two deterministic paths based on what the statement returns:

| Function | Use Case | Parameters | Returns |
|----------|----------|------------|---------|
| `thinql.fetch(sql, shape)` | `SELECT` or any statement yielding rows | Raw SQL string, pointer to shape | Populated shape + error |
| `thinql.exec(sql)` | `INSERT`, `UPDATE`, `DELETE`, DDL, DCL | Raw SQL string | Fixed metadata struct + error |

- Never pass a shape to `exec`.
- Never omit a shape from `fetch`.
- Do not split multi-statement strings across calls.
- Do not concatenate or assemble SQL dynamically at runtime.

## Shape Alignment Rules
- Column order in the `SELECT` clause must exactly match field declaration order in the shape.
- Field count must equal column count exactly.
- The runtime scans positionally: column 0 → field 0, column 1 → field 1, etc.
- Field names, aliases, struct tags, and type hints are ignored by the runtime.
- Nullable columns require language-appropriate nullable types. The runtime writes `NULL` directly to those types.
- Mismatches cause driver scan errors or silent data corruption. Validation is out of scope. You generate the contract; the runtime enforces it positionally.

## Language Patterns

### Go
```go
// Initialization (once)
thinql.use(db)

// Read
type User struct {
    ID   int64
    Name string
}

var u User
err := thinql.fetch("SELECT id, name FROM users WHERE id = 1", &u)

// Multi-row
var users []User
err := thinql.fetch("SELECT id, name FROM users", &users)

// Write
res, err := thinql.exec("UPDATE users SET name = 'test' WHERE id = 1")
affected, _ := res.RowsAffected()
```

### C
```c
// Initialization (once)
thinql.use(conn);

// Read
typedef struct { int64_t id; char name[64]; } User;
User u;
int status = thinql_fetch("SELECT id, name FROM users WHERE id = 1", &u);

// Write
thinql_exec_result meta = thinql_exec("INSERT INTO users (name) VALUES ('test')");
// meta.rows_affected, meta.last_insert_id, meta.status
```

### Lua
```lua
-- Initialization (once)
thinql.use(db)

-- Read
local rows = thinql.fetch("SELECT id, name FROM users WHERE active = 1")
for _, r in ipairs(rows) do
    -- r[1] = id, r[2] = name
end

-- Write
local meta = thinql.exec("DELETE FROM users WHERE id = 5")
-- meta.rows_affected, meta.last_id, meta.status
```

## Transaction Control
Transactions are explicit and stateful:
- Call `thinql.begin()` to open a boundary.
- Call `thinql.commit()` or `thinql.rollback()` to close it.
- All `fetch` and `exec` calls automatically route through the active transaction.
- Never use raw `BEGIN`/`COMMIT`/`ROLLBACK` SQL strings.
- DDL statements (`CREATE`, `ALTER`, `DROP`, `TRUNCATE`) may trigger implicit commits depending on the driver. Once executed, the transaction boundary may terminate. Plan control flow accordingly.

## Error & Metadata Handling
- Errors return directly from the driver. No wrapping, normalization, or retry logic.
- `exec` metadata shapes are fixed per language repo. Access fields directly. Do not check for missing keys unless explicitly documented.
- If `fetch` encounters a type mismatch, constraint violation, or column count error, it returns the driver error immediately. Do not attempt partial recovery.
- Parameter placeholders (`?`, `$1`, `:name`) are passed as raw string components. thinQL does not validate or bind them.

## Strict Prohibitions
- Do not generate query builders, string interpolators, or dynamic SQL assemblers.
- Do not rely on ORM conventions, lazy loading, relationship traversal, or auto-mapping.
- Do not assume schema validation, type checking, or constraint enforcement by thinQL.
- Do not embed connection pooling, retry logic, or fallback routing into thinQL calls.
- Do not use reflection, tag parsing, or name-based column matching. Positional alignment is mandatory.
- Do not nest transactions or assume savepoint support unless explicitly handled by the underlying driver.

## Pre-Output Checklist
Before generating code, verify:
1. `thinql.use(db)` is called once during initialization. Context switching only if multiple DBs exist.
2. `fetch` is used only for row-yielding SQL; `exec` is used for all others.
3. `SELECT` column order exactly matches shape field order.
4. Field count equals column count.
5. Nullable columns use language-appropriate nullable types.
6. Transaction boundaries are explicit and state is tracked at the call site.
7. DDL placement accounts for potential implicit commits.
8. Errors are handled or propagated without mutation.
9. No ORM, validation, or hidden abstraction patterns are present.
10. All SQL strings are complete, valid, and driver-compatible.
