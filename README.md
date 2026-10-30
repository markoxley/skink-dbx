# skink-dbx

Shared database-driver toolkit for Skink: the common foundation every
wire-protocol driver builds on. It provides the `Driver`/`Conn`/`Tx`
abstraction (Skink has no interfaces, so the vtable is expressed as
function-valued struct fields), the `Value`/`Rows` data model, DSN
parsing, a buffered transport with in-place TLS upgrade, SCRAM
authentication, and a bounded connection pool.

Wire drivers — `skink-postgres`, `skink-mysql`, `skink-mssql`,
`skink-mongo`, `skink-redis` — each register a `dbx.Driver` under their
DSN scheme. Applications then open connections without ever naming a
driver:

```skink
import "mod/markoxley/skink-dbx"

conn, err := dbx.Open("postgres://alice:secret@db.internal:5432/shop?sslmode=require")
if str.Len(err.message) > 0 {
    // handle error
}
rows, qerr := conn.Query("select id, name from users where id = $1", args)
for rows.Next() {
    id := rows.Int64("id")
    name := rows.String("name")
}
conn.Close()
```

## Modules

| module        | contents |
|---------------|----------|
| `dbx`         | root module — re-exports, `Open`, `OpenPool`, `RegisterDriver` |
| `dbx_dsn`     | `ConnInfo`, `ParseDSN` — `scheme://user:pass@host:port/db?k=v` URLs, percent-decoding, IPv6 literals |
| `dbx_value`   | `Value` tagged union — NULL/int/float/text/blob/bool/time, binary-safe (explicit lengths, NUL-safe) |
| `dbx_rows`    | `Rows` eager result set — `Next`/`Get`/`ByName`/`Int64`/`String` |
| `dbx_driver`  | `Driver`/`Conn`/`Tx` fn-vtables, scheme registry, `Open` dispatch |
| `dbx_bufconn` | `BufConn` buffered transport — `ReadN`/`ReadUntil`/`PeekN`/`ReadByte`, `StartTLS` upgrade |
| `dbx_scram`   | SCRAM-SHA-256 / SCRAM-SHA-1 client (RFC 5802/7677), binary-safe HMAC/PBKDF2 |
| `dbx_pool`    | `Pool` — bounded acquire/release with condition-variable blocking |

## Installation

dbx is distributed through `scales`, the Skink package manager. Inside a
project directory (one containing a `skink.toml` — create one with
`scales init` if needed):

```sh
scales add mod/markoxley/skink-dbx
scales install
```

then:

```skink
import "mod/markoxley/skink-dbx"
```

## Writing a driver

A driver registers itself under its DSN scheme and fills in the `Conn`
vtable:

```skink
import "mod/markoxley/skink-dbx"

fn myOpen(info: dbx.ConnInfo) -> (dbx.Conn, errors.Error) {
    // dial, authenticate, return a Conn with the fn fields set
    ...
}

fn register() {
    dbx.RegisterDriver(dbx_driver.Driver{Name: "mydb", Open: myOpen})
}
```

`dbx_bufconn.BufConn` handles the socket lifecycle: `Dial`, then
(optionally) `StartTLS`, then `ReadN`/`ReadUntil`/`WriteAll` for the wire
codec. SCRAM auth — used by PostgreSQL and MongoDB — is a three-step
exchange through `dbx_scram.Scram`.

## Testing

Run the whole suite with the bundled harness (it locates the skink repo
or an installed toolchain automatically):

```sh
./run_tests.sh
```

Suites covered:

- `dsn_test.skink` — DSN parsing, percent-decoding, IPv6, error cases
- `value_test.skink` — Value model incl. NUL-containing blobs
- `rows_test.skink` — result-set iteration and typed accessors
- `driver_test.skink` — vtable dispatch + scheme registry (fake driver)
- `scram_test.skink` — RFC 7677 / RFC 5802 oracle vectors
- `pool_test.skink` — bounded acquire/release, blocking, close
- `bufconn_test.skink` — ReadN/PeekN/ReadUntil over a live loopback socket

`install_local.sh` verifies `scales` installability end-to-end against a
local mirror (no GitHub push needed): copies the package into
`$SCALES_GITHUB_MIRROR`, then `scales add` + compile in a scratch project.

## License

Apache-2.0
