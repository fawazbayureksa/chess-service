# Chess Game

A backend-first chess platform written in Go. It supports **human vs human (online multiplayer)** and **human vs computer**. All game logic lives on the server, so a web UI and mobile apps can be added later as thin clients on the same API.

> **Status:** Planning / early development. Everything below describes the target design. Items marked *(proposed)* are decisions that can still change.

---

## Table of Contents

- [Goals](#goals)
- [Features](#features)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [API Overview](#api-overview)
- [Getting Started](#getting-started)
- [Configuration](#configuration)
- [Testing](#testing)
- [Monitoring and Admin](#monitoring-and-admin)
- [Roadmap](#roadmap)
- [Licensing Notes](#licensing-notes)

---

## Goals

- One backend that is the **single source of truth** for rules, clocks, and results.
- Same API contract for web and mobile, so no feature is built twice.
- Play vs computer with adjustable difficulty, powered by an open source engine.
- Realtime multiplayer with reliable reconnect and server-side clocks.
- Simple to deploy: a single Go binary per service behind Nginx.

## Features

**Core**
- Full chess rules: legal moves, check, checkmate, stalemate, castling, en passant, promotion, threefold repetition, 50-move rule
- FEN and PGN import/export
- Move history and game replay

**Play modes**
- Human vs computer (difficulty levels)
- Human vs human via invite link
- Quick match queue by time control and rating *(later phase)*

**Realtime**
- Live moves, clocks, resign, draw offer/accept, rematch, chat
- Reconnect with full state recovery

**Accounts and rating**
- Register/login, guest play, JWT with refresh tokens
- Elo (or Glicko-2) rating and leaderboard *(later phase)*

## Architecture

```
      Web (Next.js)        Mobile
            \                /
             REST + WebSocket
                    |
                  Nginx
                    |
        +-----------+------------+
        |      Go services       |
        |  api   | realtime      |
        |  engine workers        |
        +-----+----------+-------+
              |          |
            MySQL      Redis
         (users, games, (live state, queues,
           moves)        pub/sub, matchmaking)
                         |
                   Stockfish processes
                   (UCI over stdin/stdout)
```

**Design rules**
- The server validates **every** move. Clients never decide legality, results, or time.
- The computer opponent runs server-side. The client sends a move and receives the engine reply as an event.
- Clocks are server-side: store timestamps and remaining time, and fire a delayed check at the expected flag time.
- Move requests are idempotent (the client sends the move number), because mobile networks retry.
- The game state shape is shared by REST and WebSocket: `fen`, `moves[]`, `turn`, `clocks`, `status`, `result`.

## Tech Stack

| Area | Choice |
|---|---|
| Language | Go |
| HTTP router | Chi (`go-chi/chi`) |
| WebSocket | `gorilla/websocket` or `coder/websocket` *(proposed)* |
| Chess rules | A Go chess library such as `notnil/chess` *(verify maintenance and completeness, wrap behind an interface)* |
| Engine | Stockfish over UCI |
| Database | MySQL |
| DB access / migrations | `sqlc` + `goose` *(proposed)* |
| Cache, queues, pub/sub | Redis |
| Auth | JWT access + refresh tokens |
| API docs | OpenAPI (swaggo or a spec-first approach) |
| Metrics | Prometheus + Grafana |
| Deployment | Docker, Nginx (with WebSocket upgrade), systemd or Docker on VPS/GCP |

## Project Structure

*(proposed)*

```
.
├── cmd/
│   ├── api/            # REST + WebSocket server entrypoint
│   └── worker/         # engine and background workers (optional split)
├── internal/
│   ├── auth/           # register, login, tokens, guests
│   ├── game/           # game lifecycle, move handling, results
│   ├── chess/          # rules interface + implementation wrapper
│   ├── engine/         # Stockfish process pool, UCI client, difficulty
│   ├── clock/          # server-side clocks and timeout handling
│   ├── realtime/       # WebSocket hub, rooms, events
│   ├── matchmaking/    # queue and pairing
│   ├── rating/         # Elo / Glicko-2
│   ├── store/          # database access (sqlc generated)
│   └── config/         # env and config loading
├── migrations/         # SQL migrations
├── api/                # OpenAPI spec
├── deploy/             # Dockerfile, compose, nginx, systemd units
├── docs/
└── README.md
```

## API Overview

All endpoints are versioned under `/v1`. This is the planned contract.

**REST**

| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/v1/auth/register` `/login` `/refresh` `/guest` | Authentication |
| POST | `/v1/games` | Create game (`mode`: `ai` or `pvp`, `timeControl`, `aiLevel`, `color`) |
| GET | `/v1/games/{id}` | Full game state |
| POST | `/v1/games/{id}/moves` | Make a move |
| POST | `/v1/games/{id}/resign` | Resign |
| POST | `/v1/games/{id}/draw/offer` `/draw/accept` | Draw handling |
| GET | `/v1/games/{id}/pgn` | Export PGN |
| GET | `/v1/users/me/games` | Game history |
| POST / DELETE | `/v1/matchmaking/queue` | Join / leave queue |

**WebSocket events**

- Server to client: `game.state`, `move.made`, `clock.update`, `game.ended`, `draw.offered`, `opponent.disconnected`
- Client to server: `game.join`, `move`, `chat`

## Getting Started

> The project is not implemented yet. This section will be filled in as the first milestone lands.

Planned prerequisites:

- Go (current stable version)
- Docker and Docker Compose (MySQL, Redis)
- Stockfish binary available on the server `PATH` (or path set in config)

Planned flow:

```bash
# 1. Clone
git clone <repository-url>
cd chess-game

# 2. Start dependencies
docker compose up -d

# 3. Run migrations
make migrate-up

# 4. Run the API
make run
```

## Configuration

Configuration is read from environment variables *(proposed names)*:

| Variable | Description |
|---|---|
| `APP_ENV` | `development` or `production` |
| `HTTP_ADDR` | Listen address, for example `:8080` |
| `DB_DSN` | MySQL connection string |
| `REDIS_ADDR` | Redis address |
| `JWT_SECRET` | Signing secret for tokens |
| `STOCKFISH_PATH` | Path to the Stockfish binary |
| `ENGINE_POOL_SIZE` | Number of concurrent engine processes (match CPU cores) |
| `ENGINE_MOVE_TIMEOUT` | Max time per engine move |

Settings that admins change at runtime (maintenance mode, difficulty limits, matchmaking rules) will be stored in the database and cached.

## Testing

Planned test layers:

- **Rules:** unit tests with known positions and PGNs (castling, en passant, promotion, stalemate, repetition, 50-move rule)
- **API:** integration tests for the full game lifecycle
- **Realtime:** two simulated WebSocket clients playing a full game, including disconnect and reconnect
- **Engine:** timeout handling and automatic restart of a crashed Stockfish process
- **Load:** many concurrent games plus engine jobs (k6 or similar)

```bash
go test ./...
```

## Monitoring and Admin

- **Metrics:** expose `/metrics` (Prometheus). Track active games, engine queue depth, move latency, WebSocket connections.
- **Dashboards:** Grafana.
- **Uptime checks:** Uptime Kuma or similar.
- **Admin UI:** a separate Next.js admin app calling the Go API, to be built after the core game flow works. A minimal server-rendered admin can come first if needed.

## Roadmap

1. **Setup:** repository, Docker Compose, config, CI, OpenAPI skeleton
2. **Rules layer:** `ChessRules` interface, implementation, heavy tests
3. **Game API:** create game, move, resign, draw, PGN export
4. **Play vs computer:** Stockfish pool, difficulty levels, engine move flow
5. **Auth and persistence:** users, guests, JWT, history
6. **Multiplayer realtime:** WebSocket rooms, server clocks, reconnect
7. **Matchmaking and rating:** queue, Elo or Glicko-2, leaderboard
8. **Hardening:** rate limiting, anti-cheat heuristics, load tests
9. **Clients:** web UI (Next.js), then mobile with push notifications
10. **Admin and monitoring UI**

## Licensing Notes

Stockfish and several chess libraries are licensed under the **GPL**. Running Stockfish as a separate process on the server and talking to it over UCI is the usual approach, but license obligations depend on how you distribute your software. Review each dependency's license before any commercial or closed-source release, and get legal advice if needed.

The project's own license is to be decided.
