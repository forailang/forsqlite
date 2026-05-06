# forsqlite

SQLite library for forai. Wraps libsqlite3 via FFI and exposes a clean, typed API.

## Language Reference

See [forai language.md](https://github.com/forailang/forai/blob/main/language.md)
for full forai syntax and features.

## Public API

```fai
use { open, close, exec, exec_params, query, query_params, last_insert_rowid, migrate, version } from Forsqlite

# Open / close
let db = open(':memory:')      # or a file path
close(db)

# Execute DDL / DML
exec(db, 'CREATE TABLE ...')
exec_params(db, 'INSERT INTO t VALUES (?)', [value])

# Query → Dictionary[] (rows keyed by column name; types preserved)
let rows = query(db, 'SELECT * FROM t')
let rows = query_params(db, 'SELECT * FROM t WHERE x > ?', [min])

# After an INSERT, retrieve the assigned id (works for any table with
# an INTEGER PRIMARY KEY column).
exec_params(db, 'INSERT INTO tasks (title) VALUES (?)', [title])
let id = last_insert_rowid(db)

# Apply pending .sql migrations in lex order. Records each filename in
# `_fai_migrations` so subsequent calls only run new files.
let applied_count = migrate(db, 'db/migrations')

# Metadata
version()   # → "3.x.x"
```

## Error Handling

Every function throws on a SQLite error (the message includes
`sqlite3_errmsg()` output). Wrap calls in `try ... catch` when you
need user-facing recovery:

```fai
try
  exec_params(db, 'INSERT INTO users (email) VALUES (?)', [email])
catch e
  # `e` is the thrown Error; surface it, log it, retry, etc.
end
```

For inner-loop work where any failure should fast-fail, just let the
exception propagate.

## Transactions

Use raw SQL — there is no Rust-style RAII wrapper. Pair a `BEGIN`
with `COMMIT` on the success path and `ROLLBACK` in `catch`:

```fai
exec(db, 'BEGIN')
try
  exec_params(db, 'UPDATE accounts SET balance = balance - ? WHERE id = ?', [amount, from_id])
  exec_params(db, 'UPDATE accounts SET balance = balance + ? WHERE id = ?', [amount, to_id])
  exec(db, 'COMMIT')
catch e
  exec(db, 'ROLLBACK')
  throw e
end
```

`migrate()` uses this same pattern internally — see `src/migrate.fai`.

## File Structure

```
src/
  _ffi.fai              — sqlite3 extern block and result code constants (all private, loads first)
  types.fai             — Connection type
  open.fai              — open()
  close.fai             — close()
  exec.fai              — exec()
  exec_params.fai       — exec_params()
  query.fai             — query()
  query_params.fai      — query_params()
  last_insert_rowid.fai — last_insert_rowid()
  migrate.fai           — migrate() — apply pending *.sql files, track in _fai_migrations
  version.fai           — version()
  internal.fai          — collect_rows(), bind_params(), bool_to_int() private helpers

tests/
  migrations/           — fixture .sql files exercised by migrate's happy-path tests
  migrations_bad/       — fixture with a deliberately broken .sql file for the rollback test
```

## Key Design Notes

- `private:` is a sticky mode in forai — once declared in a file, all subsequent
  declarations in that file become private. `_ffi.fai` and `internal.fai` use this
  to keep SQLite internals out of the public API.

- `_ffi.fai` is prefixed with `_` so it sorts before all other files alphabetically.
  This matters because `let` constants (SQLITE_OK, etc.) are not forward-declared by
  the checker — they must appear before any file that references them.

- Column types are preserved: `INTEGER` → `Int`, `REAL` → `Float`, `TEXT` → `String`, `NULL` → `null`.

- Params in `exec_params` / `query_params` are bound via `?` placeholders and dispatched
  to `sqlite3_bind_int`, `sqlite3_bind_double`, `sqlite3_bind_text`, or `sqlite3_bind_null`
  based on runtime type. `Bool` values bind as `INTEGER` 0/1, matching SQLite's convention
  (no separate boolean storage class).

- `migrate(db, dir)` is a single-transaction apply: BEGIN, exec each pending file in lex
  order, INSERT into `_fai_migrations`, COMMIT. A failure mid-pass triggers ROLLBACK and
  the error propagates so partial schema state never lands in the database.

## Roadmap

- **Typed row projection.** A future `query_typed<T>(db, sql, params) -> T[]`
  will project rows directly into a record type via `from_dict`. Held until
  the forai compiler supports `@type T` + `from_dict(d)` in the same body.
  For now, project manually from `query_params`'s `Dictionary[]` rows.

## Requirements

libsqlite3 must be installed (`apt install libsqlite3-dev` / `brew install sqlite`).
