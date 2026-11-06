# agent.md

Instructions for AI coding agents working on this repository. Read this file and `README.md` before making changes.

## Project summary

A backend-first chess platform written in Go. It supports online multiplayer and play vs computer (Stockfish). The server is the single source of truth for rules, clocks, and results. Web and mobile clients are thin and come later.

**Current status:** planning / early development. The skeleton may not exist yet. Check the repository before assuming a package or command exists, and do not invent files, endpoints, or commands that are not in the repo.

## Stack

| Area | Choice |
|---|---|
| Language | Go (current stable) |
| HTTP router | Chi (`github.com/go-chi/chi/v5`) |
| WebSocket | `gorilla/websocket` or `coder/websocket` (one library only, decide once) |
| Chess rules | Go chess library behind the `chess.Rules` interface |
| Engine | Stockfish over UCI (separate process) |
| Database | MySQL, with `sqlc` and `goose` migrations |
| Cache, queues, pub/sub | Redis |
| Auth | JWT access + refresh tokens |
| Metrics | Prometheus |

Do not add a new framework or major dependency without asking. Prefer the standard library when it is enough.

## Architecture rules (do not break these)

1. **Server authority.** Every move is validated on the server. Never trust the client for legality, results, clock values, or turn order.
2. **Rules behind an interface.** All chess logic goes through the `chess.Rules` interface (`LegalMoves`, `ApplyMove`, `Status`, FEN/PGN helpers). No other package imports the underlying chess library directly.
3. **Server-side clocks.** Store timestamps and remaining time. Detect timeouts with delayed checks, not client reports.
4. **Idempotent moves.** A move request carries the expected move number. Reject duplicates and out-of-order requests.
5. **One state shape.** REST and WebSocket return the same game state: `fen`, `moves[]`, `turn`, `clocks`, `status`, `result`.
6. **Engine isolation.** Stockfish runs as a separate process managed by a pool, with per-move timeouts, a concurrency limit, and automatic restart on crash. Never block HTTP handlers on engine work without a timeout.
7. **Versioned API.** All routes live under `/v1`. Do not make breaking changes to an existing version.

## Project layout (proposed)

```
cmd/api/           entrypoint: REST + WebSocket
cmd/worker/        engine and background workers (optional split)
internal/auth/     register, login, tokens, guests
internal/game/     game lifecycle, move handling, results
internal/chess/    Rules interface + implementation wrapper
internal/engine/   Stockfish pool, UCI client, difficulty
internal/clock/    server-side clocks and timeouts
internal/realtime/ WebSocket hub, rooms, events
internal/matchmaking/
internal/rating/
internal/store/    DB access (sqlc generated)
internal/config/   env config
migrations/        SQL migrations
api/               OpenAPI spec
deploy/            Dockerfile, compose, nginx, systemd
```

Keep business logic in `internal/`. Keep `cmd/` thin (wiring only). Avoid circular imports; depend on interfaces at package boundaries.

## Go conventions

- Format with `gofmt` (or `goimports`); run `go vet` and the linter (`golangci-lint`) before finishing.
- Pass `context.Context` as the first parameter to anything that does I/O, and respect cancellation.
- Return errors; wrap with `fmt.Errorf("...: %w", err)`. No `panic` in request paths.
- Define sentinel or typed errors for domain failures (illegal move, wrong turn, game over) and map them to HTTP status codes in one place.
- Handlers are plain `http.HandlerFunc` or `http.Handler` (Chi compatible). Use Chi route groups and standard `net/http` middleware.
- Validate and bind input at the edge (handlers), not deep in domain code.
- Use structured logging (`log/slog`) with request and game IDs. Never log secrets, tokens, or passwords.
- Keep goroutines owned and stoppable: every goroutine has a clear owner, exits on context cancel, and does not leak. Protect shared state with channels or mutexes, and run tests with `-race`.
- Small functions, clear names, comments that explain why rather than what.

## Database

- Schema changes only through new migration files in `migrations/`. Never edit an applied migration.
- Write SQL queries for `sqlc`, then regenerate. Do not hand-edit generated code.
- Use transactions where a move and the game state must change together.
- Live game state may be cached in Redis, but MySQL is the durable record.

## Security

- Hash passwords with bcrypt or argon2. Keep JWT secrets and DB credentials in environment variables only.
- Apply rate limiting to auth and move endpoints.
- Validate all input sizes and formats. Treat PGN/FEN from clients as untrusted and validate them.
- Do not expose internal errors to clients.

## Testing

- Every change ships with tests. Run `go test -race ./...` before declaring work done.
- **Rules tests:** known positions and PGNs covering castling, en passant, promotion, stalemate, threefold repetition, 50-move rule, insufficient material.
- **Integration tests:** full game lifecycle through the HTTP API.
- **Realtime tests:** two WebSocket clients playing a game, including disconnect and reconnect.
- **Engine tests:** timeout and crash recovery. Use a fake UCI engine in unit tests so they do not depend on Stockfish being installed.
- Prefer table-driven tests.

## Commands

These are the intended commands. Check the `Makefile` before using them, and add any that are missing instead of guessing.

```bash
make run           # run the API locally
make test          # go test -race ./...
make lint          # golangci-lint run
make migrate-up    # apply migrations
make migrate-down  # roll back one migration
make sqlc          # regenerate DB code
docker compose up -d   # MySQL + Redis for local development
```

## Workflow for agents

1. Read `README.md` and this file. Inspect the actual repository state first.
2. Make small, focused changes. One concern per change.
3. Follow the roadmap order in `README.md` unless told otherwise.
4. Write or update tests with the code. Run formatting, vet, lint, and tests.
5. Update `README.md` and the OpenAPI spec when behavior or endpoints change.
6. If a requirement is ambiguous or conflicts with the architecture rules above, ask instead of guessing.
7. Do not commit secrets, `.env` files, or generated binaries.
8. Summarize what changed and what was verified when you finish.

## Out of scope for now

Web UI, mobile apps, and the admin app come after the core game flow works. Do not build them in this repository unless asked. The admin UI is planned as a separate Next.js app calling this API.

## Licensing note

Stockfish and some chess libraries are GPL. Keep Stockfish as a separate process. Check a dependency's license before adding it, and flag anything that could affect closed-source use.
