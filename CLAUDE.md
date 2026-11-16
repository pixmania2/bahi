# CLAUDE.md

Guidance for Claude Code in this repository.

## What this repository is
Bahi is P1 of the Money Stack: a double-entry ledger service with bank reconciliation, written in Go, backed by PostgreSQL 16 (TigerBeetle as an alternative engine later). It is the system of record for the whole stack and also the engine inside the simulated bank, RTGS book and depository (see the platform repo `money-stack-platform`).

The owner is learning Go with this project. Prefer clear, idiomatic, standard-library-first Go over clever code, and explain non-obvious Go idioms in PR descriptions.

## Commands (create these targets as the repo grows)
- `make up`: start Postgres (and later Redpanda) via the platform docker-compose
- `make test`: unit + property + integration tests (`go test ./... -race`)
- `make lint`: `golangci-lint run`
- `make migrate`: run goose migrations
- `make bench`: reproduce benchmarks (P1 week 5)
- `make demo`: run the demo scenario

## Domain rules (never violate)
- Amounts are integer minor units (`int64`) with an asset code (`INR` scale 2; securities scale 0). Never floats.
- Every posted entry balances **per asset**: sum of debits == sum of credits > 0.
- Ledger records are append-only. Corrections are reversal entries. The app DB role has no UPDATE/DELETE on entries/lines.
- Every mutating API requires an `Idempotency-Key`; same key + same body → same response; same key + different body → 409.
- Business dates (`effective_date`) are `date` values in Asia/Kolkata; timestamps are UTC `timestamptz`.
- Entries into a closed period are rejected.
- Tenant comes from the auth token / `app.tenant_id` session setting and is enforced by Postgres RLS, never from request bodies.
- No real personal or business data anywhere. Test data is synthetic.
- Secrets in `.env` (gitignored); `.env.example` committed.

## Stack & conventions
- Go (latest stable), `net/http` + `chi`, `pgx/v5`, `sqlc`, `goose`, `testcontainers-go`, `pgregory.net/rapid` for property tests.
- Layout: `cmd/bahi/` (main), `internal/<domain>/` (money, ledger, recon, api, store), `migrations/`, `api/openapi.yaml`, `tests/`.
- Integration tests use Testcontainers Postgres, **never** mocks of the database.
- Errors: RFC 9457 problem+json. Logs: structured JSON via `log/slog`, no PII.

## How to work here
- Change the contract (`api/openapi.yaml`, migrations) first, then code.
- Write or update tests first for ledger logic; add a property test for any new invariant.
- Keep PRs small and tied to one GitHub issue; describe what changed and how it was tested.
- Ask before adding a dependency.
- **Do not write ADRs, invariants or benchmark conclusions**: the owner writes those. Draft code, tests and tooling only.
- Reference rules carried over from the owner's accounting product: `docs/reference/from-accounting-product.md`.
