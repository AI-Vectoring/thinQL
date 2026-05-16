### C
- `use(db)` accepts opaque driver handle (`sqlite3*`, `PGconn*`, etc.).
- `fetch(sql, target)` accepts `void*` to struct array or single struct. Runtime iterates columns by index, calls driver accessor functions (`sqlite3_column_xxx`, `PQgetvalue`), writes directly to struct memory offsets.
- `exec(sql)` returns fixed struct:
  ```c
  typedef struct { int64_t rows_affected; int64_t last_insert_id; int status; } thinql_exec_result;
  ```
- Memory: caller-allocated buffers or runtime allocator with explicit `thinql_free()` release rule. Implementation must document ownership model.
