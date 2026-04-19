# thinQL Go Implementation Reference

This document walks through the reference implementation function by function. Each entry follows the same structure: purpose, design rationale, annotated implementation, and code walkthrough. The implementation strictly enforces the thinQL spec: positional scanning, explicit transaction state, registry-based connection routing, and zero ORM abstraction.

---

### Function: `Add`

**Purpose**  
Register a database connection under an alias. Centralizes driver initialization, pool creation, and DSN passthrough. Eliminates the need for callers to manage raw driver handles.

**Design Rationale**  
`Add` accepts a human-readable alias, a native driver name, and a raw DSN string. It forwards the DSN directly to `sql.Open()`, letting the native driver handle parsing, pooling, and connection lifecycle. The registry caches the resulting `*sql.DB` handle. If it's the first registration, it automatically sets the active context, making `Use()` optional for single-DB projects. No DSN normalization or config parsing occurs.

**Implementation**
```go
var registry = make(map[string]*sql.DB)
var activeAlias string

// Add registers a database connection. Driver and DSN are passed directly to the native driver.
func Add(alias, driver, dsn string) error {
    db, err := sql.Open(driver, dsn)
    if err != nil {
        return fmt.Errorf("thinql: driver init failed for %s: %w", alias, err)
    }
    registry[alias] = db
    // Auto-set default context on first registration
    if activeAlias == "" {
        activeAlias = alias
    }
    return nil
}
```

**Walkthrough**  
`registry` stores active connections by alias. `sql.Open()` is called with zero modification to the DSN, preserving native driver behavior (pool defaults, timeout flags, etc.). The handle is stored for later routing. The `activeAlias == ""` check ensures that if only one database is registered, all subsequent calls automatically target it without requiring an explicit `Use()` call. Errors bubble up immediately with no retry or fallback logic.

---

### Function: `Use`

**Purpose**  
Switch the active execution context to a registered database alias.

**Design Rationale**  
`Use` is only required in multi-database codebases. It validates that the alias exists in the registry and updates the active context pointer. It does not open new connections or verify liveness. Connection state is entirely managed by the native pool. This keeps the function stateless and O(1).

**Implementation**
```go
// Use sets the active database context for subsequent Fetch, Exec, and transaction calls.
func Use(alias string) error {
    if _, ok := registry[alias]; !ok {
        return fmt.Errorf("thinql: unknown database alias: %s", alias)
    }
    activeAlias = alias
    return nil
}
```

**Walkthrough**  
A direct map lookup confirms the alias exists. If valid, `activeAlias` is overwritten. All execution functions reference `activeAlias` to resolve the correct `*sql.DB` handle. No connection verification is performed; dead connections are handled by `database/sql`'s internal retry/pool logic during execution.

---

### Function: `Fetch`

**Purpose**  
Execute row-yielding SQL (`SELECT`, etc.) and scan results directly into an LLM-defined shape.

**Design Rationale**  
`Fetch` enforces the 1:1 positional contract. It accepts any pointer target, uses reflection strictly to gather field addresses in declaration order, and passes them to `rows.Scan()`. No struct tags, field names, or type coercion are used. It branches based on whether the target is a single struct or a slice, routing to appropriate scan helpers. Errors from the driver or scan mismatches are returned immediately.

**Implementation**
```go
// Fetch executes a row-yielding query and scans positionally into the provided shape.
// target must be a pointer to a struct or a pointer to a slice of structs.
func Fetch(query string, target any) error {
    conn, err := getConn()
    if err != nil {
        return err
    }

    rows, err := conn.Query(query)
    if err != nil {
        return err
    }
    defer rows.Close()

    val := reflect.ValueOf(target)
    if val.Kind() != reflect.Ptr || val.IsNil() {
        return fmt.Errorf("thinql: target must be a non-nil pointer")
    }
    elem := val.Elem()

    if elem.Kind() == reflect.Slice {
        return scanSlice(rows, elem)
    }
    return scanSingle(rows, elem)
}
```

**Walkthrough**  
`getConn()` resolves the active `*sql.DB` or `*sql.Tx`. `conn.Query()` executes the raw string. The target is validated as a non-nil pointer. Reflection inspects only the type kind: if it's a slice, iteration begins; otherwise, a single row is scanned. The reflection step extracts exactly `N` field pointers in declaration order and feeds them to `rows.Scan()`. No name matching occurs. If column count ≠ field count, or types mismatch, `database/sql` returns a hard error. The spec intentionally leaves recovery to the call site.

---

### Function: `Exec`

**Purpose**  
Execute non-row statements (`INSERT`, `UPDATE`, `DELETE`, DDL, DCL) and return raw driver metadata.

**Design Rationale**  
`Exec` requires no target shape. It routes through the active context, executes the SQL, and returns `sql.Result` directly. The LLM or calling code extracts `.RowsAffected()` or `.LastInsertId()` as needed. This avoids wrapper objects, preserves driver-specific behavior, and keeps the return type flat.

**Implementation**
```go
// Exec executes a non-row statement and returns the raw driver result interface.
func Exec(query string) (sql.Result, error) {
    conn, err := getConn()
    if err != nil {
        return nil, err
    }
    return conn.Exec(query)
}
```

**Walkthrough**  
Identical routing to `Fetch`, but calls `conn.Exec()`. The `sql.Result` interface is returned unchanged. This means SQLite, PostgreSQL, and MySQL quirks (like `LastInsertId` returning `0` or `-1` unsupported) surface directly. thinQL does not normalize, wrap, or guess metadata shapes.

---

### Function: `Begin`, `Commit`, `Rollback`

**Purpose**  
Provide explicit, stateful transaction boundaries without ORM session management.

**Design Rationale**  
Transactions are tracked at the package level via `activeTx`. `Begin` opens a transaction on the currently active database. `Commit` and `Rollback` finalize it and clear the state. `getConn()` prioritizes `activeTx` over the base `*sql.DB`, ensuring all `Fetch` and `Exec` calls automatically route through the open transaction. Nested transactions and savepoints are explicitly unsupported to prevent hidden complexity.

**Implementation**
```go
var activeTx *sql.Tx

// Begin opens an explicit transaction on the active database context.
func Begin() error {
    conn, err := getConn()
    if err != nil {
        return err
    }
    db, ok := conn.(*sql.DB)
    if !ok {
        return fmt.Errorf("thinql: transaction already active")
    }
    tx, err := db.Begin()
    if err != nil {
        return err
    }
    activeTx = tx
    return nil
}

// Commit finalizes and closes the active transaction.
func Commit() error {
    if activeTx == nil {
        return fmt.Errorf("thinql: no active transaction to commit")
    }
    err := activeTx.Commit()
    activeTx = nil
    return err
}

// Rollback aborts and closes the active transaction.
func Rollback() error {
    if activeTx == nil {
        return fmt.Errorf("thinql: no active transaction to rollback")
    }
    err := activeTx.Rollback()
    activeTx = nil
    return err
}
```

**Walkthrough**  
`Begin` checks if a transaction is already open by attempting to type-assert the active connection. If it's already a `*sql.Tx`, it errors out. Otherwise, it calls `db.Begin()` and stores the handle. `Commit` and `Rollback` check for `nil` state, execute the native method, and explicitly clear `activeTx`. This guarantees a single-active-transaction model per process scope. `getConn()` (below) ensures all execution functions respect this state.

---

### Internal Helpers

**Purpose**  
Handle connection routing and strict positional scanning mechanics.

**Design Rationale**  
These functions are not exposed to the LLM or caller. `getConn` abstracts DB vs TX routing. `fieldPointers` guarantees declaration-order address extraction. `scanSingle` and `scanSlice` handle cursor iteration without ORM overhead.

**Implementation**
```go
// getConn returns the active transaction or falls back to the base database pool.
func getConn() (interface {
    Query(string, ...any) (*sql.Rows, error)
    Exec(string, ...any) (sql.Result, error)
}, error) {
    if activeAlias == "" {
        return nil, fmt.Errorf("thinql: no database context set (call Use or Add first)")
    }
    if activeTx != nil {
        return activeTx, nil
    }
    db, ok := registry[activeAlias]
    if !ok {
        return nil, fmt.Errorf("thinql: context alias lost: %s", activeAlias)
    }
    return db, nil
}

// scanSingle scans exactly one row into the target struct fields by position.
func scanSingle(rows *sql.Rows, target reflect.Value) error {
    if !rows.Next() {
        return rows.Err()
    }
    return rows.Scan(fieldPointers(target)...)
}

// scanSlice iterates rows, allocates new struct instances, and appends them.
func scanSlice(rows *sql.Rows, sliceVal reflect.Value) error {
    elemType := sliceVal.Type().Elem()
    for rows.Next() {
        newElem := reflect.New(elemType)
        if err := rows.Scan(fieldPointers(newElem.Elem())...); err != nil {
            return err
        }
        sliceVal.Set(reflect.Append(sliceVal, newElem.Elem()))
    }
    return rows.Err()
}

// fieldPointers collects pointers to struct fields in strict declaration order.
// Ignores names, tags, and alignment. Pure positional mapping.
func fieldPointers(v reflect.Value) []any {
    n := v.NumField()
    dest := make([]any, n)
    for i := 0; i < n; i++ {
        dest[i] = v.Field(i).Addr().Interface()
    }
    return dest
}
```

**Walkthrough**  
`getConn` checks `activeTx` first. If present, it returns the transaction. Otherwise, it resolves the base `*sql.DB` from the registry using `activeAlias`. This routing guarantees all queries automatically participate in the active transaction boundary.  
`scanSingle` advances the cursor once and passes positional field pointers to `rows.Scan()`.  
`scanSlice` loops through the cursor, allocates a new struct instance per row via `reflect.New`, scans into it, and appends to the slice. This matches `database/sql` idioms while preserving positional enforcement.  
`fieldPointers` is the core spec enforcer. It iterates fields by index `0..N-1`, grabs each memory address via `.Addr()`, and returns a slice of `any` pointers. Zero name resolution, zero tag parsing, zero type inference. If the LLM misaligns column order and struct field order, values write to wrong fields. The spec accepts this as a generation responsibility, not a runtime concern.
