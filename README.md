# codhoot-beam-service

Execution server for the BEAM languages — **Erlang** and **Elixir** — for
CodHoot. One service for both, because they are one toolchain: both compile to
BEAM bytecode and run on `erl`.

## Why this service exists

CodHoot's eight single-language services (`codhoot-python-service`,
`codhoot-c-service`, …) each hard-code one compiler and one entry filename, so
they cannot be reused for a language they were not built for — pointing Erlang
at the Java service would just run `javac` over an Erlang module. The backend
therefore routes Erlang and Elixir to this service through the shared runtime
image URL:

```
COMPILER_IMAGE_BEAM_SERVICE_URL=https://codhoot-beam-service-<suffix>.onrender.com
```

One variable covers both languages. (`COMPILER_FAMILY_BEAM_SERVICE_URL` works
too and covers any future BEAM-family language.) The mapping is read at backend
startup, so set the variable and restart the backend service.

## Contract

`POST /compile`, signed with
`HMAC-SHA256(secret, timestamp + path + hex(sha256(body)))` keyed by
`CODHOOT_INTERNAL_SECRET`, headers `X-CodHoot-Timestamp` (unix seconds) and
`X-CodHoot-Signature` (hex digest). Unsigned, tampered or stale requests get
401.

```json
{ "language": "erlang", "source": "-module(main).\n-export([main/0]).\nmain() -> io:format(\"hi~n\")." }
```

`GET /health` and `GET /health/live` are open — Render's health check and the
keepalive pinger must never need a signature.

| language | entry file | compile  | run                                              |
|----------|------------|----------|--------------------------------------------------|
| erlang   | `main.erl` | `erlc -o . {src}` | `erl -noshell -eval "main:main(), halt(0)."` |
| elixir   | `main.ex`  | `elixirc -o . {src}` | `elixir -e "Main.main()"`               |

Those command lines are the backend registry's own `CompileCommand` and
`RunCommand` for each language, so a job behaves identically to the local
`codhoot-runner-beam` docker path. Elixir sources are expected to define
`defmodule Main` (file `main.ex` → module `Main`), which is what the run command
invokes.

## Deployment

Render blueprint: **New → Blueprint**, point it at this repo, apply. Set
`CODHOOT_INTERNAL_SECRET` to the same value the backend uses.

The service listens on `PORT` (Render sets it to 10000; the image default is
8081). `MAX_CONCURRENT_JOBS` defaults to 2.

## Sandbox

Each job gets its own uid from a 32-entry pool, a private random 0700 working
directory, a minimal environment, rlimits applied by a trampoline re-exec, and
process-group kill on timeout. The service runs as root only to chown and
setuid; the boundary is the child. See `harden.go` / `harden_unix.go`.