# monorepo-sandbox-example

A small, synthetic monorepo for trying an AI coding agent on a database-backed
Rails bug. All code and data are made up.

```text
server/              Rails 8.1 API (the only module with real code and tests)
web/  mobile/        placeholders that give the repo a monorepo shape
services/events-svc  small Rust service that owns the events database schema
tools/               deterministic generator for the settlement test data
env/                 optional OpenHands prepared-sandbox tooling (see below)
```

## What the server needs

`server/` uses three PostgreSQL servers of different major versions:

| Logical DB | PostgreSQL | Schema owner |
|---|---|---|
| primary | 16 | Rails migrations (`db/schema.rb`) |
| ledger | 14 | Rails migrations (`db/ledger_schema.rb`) |
| events | 17 | `services/events-svc` (SQL migrations applied at startup) |

The Rails test boot needs all three, and setup waits for `events-svc` to
migrate the events databases.

## Run the tests

Needs Ruby (see `server/.ruby-version`), `libpq`, Docker with Compose, and
Python 3.

```bash
cd server
bundle install
docker compose up -d --wait
AUTH_MODE=none bin/rails dev:bootstrap   # custom task that replaces db:setup
python3 ../tools/settlements-corpus/materialize.py --only-from-tests test
bin/rails test
```

`dev:bootstrap` runs an auth gate first (`bin/auth-check`), and `AUTH_MODE`
must be set. `AUTH_MODE=none` checks no credentials. `AUTH_MODE=aws-sso` is
an example of a gate that fails closed until a person has signed in; nothing
in this repo needs AWS.

The Rust service has its own tests:

```bash
cd services/events-svc && cargo test --locked
```

## Settlement data

Some tests read settlement files. `tools/settlements-corpus` regenerates them
deterministically from `server/data/settlements/manifest.json` (731 days,
about 9.6 GB in total). The command above writes only the few days the tests
use. See `tools/settlements-corpus/README.md` for the full set.

## CI

- `ci` runs a full-history secret scan, the Rails suite against all three
  PostgreSQL versions, and the Rust tests.
- `corpus-check` (manual) regenerates all 731 days in memory, one quarter per
  job, and checks each file's size and SHA-256 against the manifest. Nothing
  is written to disk.

## Optional: prepared sandbox tooling

`env/` sketches a sandbox image built `FROM` a pinned OpenHands agent-server
release. The image carries a read-only Git seed, a pre-installed gem bundle,
service images and keyed database fixtures. `prepare-workspace --sha <commit>`
gives each task its own checkout, Compose project and privately restored
databases, and `oh-verify` re-checks a fix at exact commits. This repo's CI
does not build or test `env/`, and no prebuilt image is published.
