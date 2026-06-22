# Forsqlite

`Forsqlite` is a small SQLite wrapper for Forai. It opens SQLite connections,
executes statements, queries rows as dictionaries, binds positional parameters,
and applies ordered SQL migrations.

Import the functions directly or alias the package:

```fai
use Forsqlite as sqlite
use { getString, getInt } from std.dictionary

let db = sqlite.open(':memory:')
sqlite.exec(db, 'CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT NOT NULL, active INTEGER)')
sqlite.exec_params(db, 'INSERT INTO users (name, active) VALUES (?, ?)', ['Ada', true])

let rows = sqlite.query(db, 'SELECT id, name, active FROM users')
let id = getInt(rows[0], 'id')!
let name = getString(rows[0], 'name')!

sqlite.close(db)
```

Use `open(':memory:')` for an in-memory database or pass a filesystem path for
a persistent database. Close persistent connections with `close` when the app is
done with them.

## SQLite Extensions

Use `load_extension(db, path)` to load a SQLite extension shared library, such
as `sqlite-vec`, into an existing connection. Forsqlite enables extension
loading only for the duration of the load call and disables it afterwards.

```fai
sqlite.load_extension(db, '/path/to/vec0')
```

If the extension cannot be loaded, `load_extension` throws and the connection
remains usable for normal SQLite queries.

## Statements And Queries

Use `exec` for SQL that does not return rows, such as `CREATE TABLE`, `INSERT`,
`UPDATE`, and `DELETE`. Use `query` for `SELECT` statements.

Prefer `exec_params` and `query_params` whenever values come from variables or
user input. Parameters bind positionally to `?` placeholders.

```fai
sqlite.exec_params(db, 'INSERT INTO users (name, active) VALUES (?, ?)', ['Grace', false])

let matches = sqlite.query_params(db, 'SELECT id, name FROM users WHERE active = ?', [true])
```

Parameter binding rules:

- `null` binds as SQL `NULL`
- `Bool` binds as integer `1` or `0`
- `Int` binds as SQLite `INTEGER`
- `Float` binds as SQLite `REAL`
- other values bind as `TEXT` using string interpolation

## Reading Rows

`query` and `query_params` return `Dictionary[]`. Each row is keyed by column
name. SQLite column values are returned as Forai values:

- `INTEGER` -> `Int`
- `REAL` -> `Float`
- `TEXT` -> `String`
- `NULL` -> `null`

Use `std.dictionary` accessors to read values safely:

```fai
use { get, getString, getInt } from std.dictionary

let row = rows[0]
let id = getInt(row, 'id')!
let name = getString(row, 'name')!
let deletedAt = get(row, 'deleted_at')
```

`last_insert_rowid(db)` returns the row id from the most recent successful
insert on that connection.

## Migrations

Put `.sql` migration files in a directory and name them so lexical filename
ordering is the order you want applied:

```text
db/migrations/
  001_create_users.sql
  002_create_sessions.sql
  003_create_posts.sql
```

Then call `migrate(db, 'db/migrations')`.

```fai
let applied = sqlite.migrate(db, 'db/migrations')
print('applied ' + toString(applied) + ' migration(s)')
```

`migrate` ignores non-`.sql` files. It records applied filenames in
`_fai_migrations`, skips files already recorded, and returns the number of files
applied by the current call. The apply pass runs inside a transaction; if any
migration fails, the transaction rolls back and the error propagates.

## Errors And Transactions

Forsqlite operations throw on SQLite errors. Let the error propagate for hard
failures, or use `try ... catch` for recoverable cases such as uniqueness
violations.

```fai
try
    sqlite.exec_params(db, 'INSERT INTO users (name) VALUES (?)', [name])
catch e
    print('insert failed: ' + e.message)
end
```

Transactions are explicit SQL:

```fai
sqlite.exec(db, 'BEGIN')
try
    sqlite.exec_params(db, 'UPDATE accounts SET balance = balance - ? WHERE id = ?', [amount, fromId])
    sqlite.exec_params(db, 'UPDATE accounts SET balance = balance + ? WHERE id = ?', [amount, toId])
    sqlite.exec(db, 'COMMIT')
catch e
    sqlite.exec(db, 'ROLLBACK')
    throw e
end
```
