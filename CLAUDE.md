# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

Pterodactyl Wings — the server-side daemon for the Pterodactyl game server management panel. It manages Docker-containerized game servers via a REST API, built-in SFTP server, and WebSocket console. It communicates with the Pterodactyl Panel over HTTP.

## Build & Development Commands

```bash
make build        # Cross-compile for Linux amd64 and arm64
make debug        # Build with debug symbols and run
make rmdebug      # Start remote-debuggable session via dlv (port 2345)
make clean        # Remove build artifacts
```

```bash
go test ./...                         # Run all tests
go test -race ./...                   # Run tests with race detector (CGO_ENABLED=1)
go test ./server/... -run TestName    # Run a single test or package
```

Nix dev shell (via `direnv use flake` or `nix develop`) provides: Go 1.24, gofumpt, golangci-lint, gotools, yamlfmt.

## Architecture

### Entry Point & Startup

`wings.go` + `cmd/root.go` initialize in this order:
1. Parse config (`/etc/pterodactyl/config.yml`)
2. Set up logging with rotation
3. Initialize Docker client
4. Load all servers from disk (up to 4 in parallel via workerpool)
5. Start SFTP server and Gin HTTP API

### Core Subsystems

**Server lifecycle** (`server/`)
- `server.go` — the `Server` struct owns a Docker environment, filesystem, event emitter, websocket connections, and crash handler. Uses `sync.RWMutex` + custom `system.Locker` for power-action exclusion. Atomic bools track installation/transfer/restore in-progress states.
- `manager.go` — `Manager` is the collection of all `Server` instances. Lazy-loads from disk, persists state every minute.
- `power.go` — start/stop/restart/kill state machine.
- `install.go` — server installation and reinstallation.
- `transfer/` — server transfer between Wings nodes.

**HTTP API** (`router/`)
- Gin routes grouped under `/api/servers/:server/` (auth-gated) and a few public download/upload/websocket endpoints.
- Middleware: JWT auth (`router/middleware/`), request ID, error capture, server-existence check.
- WebSocket console lives at `/api/servers/:server/ws` (`router/websocket/`).

**Filesystem** (`server/filesystem/` + `internal/ufs/`)
- `server/filesystem/` provides archive, compression, disk quota, and chroot-style operations.
- `internal/ufs/` is a lower-level unified filesystem derived from Go stdlib with quota enforcement.

**Environment / Docker** (`environment/`)
- Abstraction over Docker containers: `IsRunning()`, `Attach()`, `SetState()`, `Send()` (console input).

**Panel communication** (`remote/`)
- HTTP client for all Panel API calls. Authenticated via JWT credentials from config.

**SFTP** (`sftp/`)
- Built-in SFTP server; activity is logged to SQLite via `internal/database/`.

**Events** (`events/`)
- Pub/sub event bus used for server state changes and console output multiplexing (`system.SinkPool`).

**Scheduled tasks** (`internal/cron/`)
- Activity log flushing and SFTP cleanup jobs via `go-co-op/gocron`.

### Key Libraries

| Library | Role |
|---------|------|
| `gin-gonic/gin` | HTTP router |
| `docker/docker` | Docker API client |
| `pkg/sftp` | SFTP server |
| `gorm.io/gorm` + SQLite | Activity log persistence |
| `apex/log` | Structured logging |
| `cobra` | CLI commands |
| `gorilla/websocket` | WebSocket console |
| `emperror.dev/errors` | Error wrapping |

### Concurrency Patterns

- Server power actions are serialized via `system.Locker` (not a plain mutex — supports context cancellation).
- Console output uses `system.SinkPool` to fan-out to multiple WebSocket clients.
- Installation/transfer/restore states are `system.AtomicBool` checked before allowing conflicting actions.
- Docker event streams and console attach use context-scoped goroutines cancelled on server stop.

## Configuration

YAML at `/etc/pterodactyl/config.yml`. The `config.Config` struct (`config/config.go`) covers:
- System paths, user/group, timezone
- API host/port, TLS, trusted proxies
- Panel URL and auth token
- Docker settings (network, pull policy, OOM killer)
- SFTP settings and remote query timeouts

Config is a process-global singleton accessed via `config.Get()`.
