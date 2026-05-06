# forsqlite

A SQLite library for [forai](https://github.com/forailang/forai). Wraps
`libsqlite3` via the forai FFI and exposes a small, typed API:

- `open` / `close` — connection lifecycle
- `exec` / `exec_params` — DDL and DML, with positional `?` parameter binding
- `query` / `query_params` — SELECT, returns `Dictionary[]` rows keyed by column name (column types preserved)
- `last_insert_rowid` — fetch the id assigned to the last `INSERT`
- `migrate` — apply pending `.sql` files from a directory in lex order, tracked in `_fai_migrations`
- `version` — linked SQLite library version

## Install

Add to your `fai.toml`:

```toml
[dependencies]
"file://../forsqlite" = "0.1.0"
```

(Path-based dependencies are the supported form today; pinning to git
URLs is on the forai roadmap.)

`libsqlite3` must be available on the host:

```bash
# Linux (Debian/Ubuntu/Arch)
sudo apt install libsqlite3-dev          # debian/ubuntu
sudo pacman -S sqlite                    # arch

# macOS
brew install sqlite
```

## Hello, world

```fai
use { open, close, exec, query, exec_params, last_insert_rowid } from Forsqlite

let db = open(':memory:')

exec(db, 'CREATE TABLE notes (id INTEGER PRIMARY KEY, body TEXT NOT NULL)')

exec_params(db, 'INSERT INTO notes (body) VALUES (?)', ['hello'])
let id = last_insert_rowid(db)

let rows = query(db, 'SELECT id, body FROM notes')
print(getString(rows[0], 'body')!)        # → 'hello'

close(db)
```

## Querying

`query` and `query_params` return `Dictionary[]`. Each row maps column
name to value, with types preserved:

| SQLite type | forai type |
| --- | --- |
| `INTEGER` | `Int` |
| `REAL` | `Float` |
| `TEXT` | `String` |
| `NULL` | `null` |

Use the typed accessors from `std.dictionary` to read columns:

```fai
let rows = query_params(db, 'SELECT id, body FROM notes WHERE id = ?', [42])
let id = getInt(rows[0], 'id')!
let body = getString(rows[0], 'body')!
```

Always bind user input through `exec_params` / `query_params` — `?`
placeholders are how you avoid SQL injection. Don't string-concat
into the SQL.

## Errors

Every function throws on a SQLite error. Wrap with `try ... catch` for
recoverable failure paths; let it propagate for hard-fail flows:

```fai
try
  exec_params(db, 'INSERT INTO users (email) VALUES (?)', [email])
catch e
  # Duplicate email, schema mismatch, etc. — handle or rethrow.
end
```

## Transactions

There's no RAII wrapper — drive `BEGIN` / `COMMIT` / `ROLLBACK`
yourself with `exec`:

```fai
exec(db, 'BEGIN')
try
  exec_params(db, 'UPDATE accounts SET balance = balance - ? WHERE id = ?', [amt, from])
  exec_params(db, 'UPDATE accounts SET balance = balance + ? WHERE id = ?', [amt, to])
  exec(db, 'COMMIT')
catch e
  exec(db, 'ROLLBACK')
  throw e
end
```

## Migrations

Drop ordered `.sql` files into a directory and call `migrate(db, dir)`.
Each filename is recorded in `_fai_migrations` so subsequent calls only
run new files. The whole apply pass is transactional — a failure
mid-pass rolls back and re-throws, so partial schema state never
lands.

```
db/migrations/
  001_create_users.sql
  002_create_sessions.sql
  003_create_posts.sql
```

```fai
let applied = migrate(db, 'db/migrations')
print('applied ' + toString(applied) + ' migration(s)')
```

## Building and testing

```bash
fai check                # format + type check
fai test                 # run all inline tests (24 tests, in-memory DB)
fai build                # compile to a runnable artifact
```

## Roadmap

- **`query_typed<T>`** — project rows directly into a record type
  (`let users User[] = query_typed(db, 'SELECT * FROM users', [])`).
  Held until the forai compiler supports `@type T` + `from_dict(d)`
  in the same function body. For now, project manually from
  `query_params`'s `Dictionary[]` rows.

## License

Apache-2.0.
