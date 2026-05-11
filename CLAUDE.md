# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

neuDrive is a personal AI identity, memory, and trust hub. It provides a central place for Claude, ChatGPT, Cursor, and other AI agents to share user identity, preferences, skills, and secrets. The codebase consists of a Go backend with embedded React frontend, supporting both self-hosted and cloud deployments.

## Tech Stack

- **Backend**: Go 1.25, chi/v5 router, JWT authentication, PostgreSQL (production) or SQLite (local dev)
- **Frontend**: React 18, TypeScript, Vite, CodeMirror for markdown editing
- **Protocols**: HTTP REST API, MCP (Model Context Protocol), OAuth 2.0 / OIDC
- **Deployment**: Docker / Docker Compose, multi-stage build

## Build Commands

```bash
# Development: start backend (:8080) + frontend dev server (:3000)
make dev

# Production build: embed frontend into Go binary
make build
# Produces: bin/neudrive, bin/neu

# Install CLI binaries to PATH
make install
# Or: ./tools/install-neudrive.sh

# Run all tests
go test ./...                          # Go tests only
cd web && npm test -- --run           # Frontend tests
go test ./internal/api -run TestHealthCheck -v   # Single Go test

# Lint
golangci-lint run                      # Requires golangci-lint installed

# Docker build
docker build -t neudrive:latest .

# Clean build artifacts
make clean
```

## Ark Server Deployment (Current)

This repository is actively deployed on the ark server (Ubuntu 24.04).

**Access URL**: http://101.126.89.206:8082
**Server Name**: ark (iv-ydz3lukphcwh2ypd7ili)
**Public IP**: 101.126.89.206

### Running with Docker Compose

```bash
cd /home/zijie/Code/neuDrive
docker compose --env-file neudrive.env up -d --build
docker compose --env-file neudrive.env restart
docker compose --env-file neudrive.env down
```

**Config file**: `neudrive.env` — must exist, never commit secrets.
Key settings:
- `PORT=8082` (8080 occupied by another service)
- `PUBLIC_BASE_URL=http://101.126.89.206:8082`
- `NEUDRIVE_LOCAL_MODE=1` (enables email/password auth)
- `NEUDRIVE_ENABLE_BILLING=0` (self-hosted, no billing UI)
- Database: PostgreSQL via Docker, port 5432

### China Network Note

The Dockerfile sets `GOPROXY=https://goproxy.cn,https://proxy.golang.com.cn,direct` for `go mod download` because the server is in mainland China and cannot reach `proxy.golang.org`.

### CLI Usage in Container

Go is NOT installed on the host. Use the binary inside the running container:

```bash
# Alias for convenience
alias neuc='docker exec -i neudrive-server-1 ./neudrive'

# Use with explicit API base and token
neuc ls --api-base http://127.0.0.1:8080 --token <token>
neuc stats --api-base http://127.0.0.1:8080 --token <token>
```

## Git Setup

| Remote | URL | Purpose |
|--------|-----|---------|
| `origin` | `https://github.com/agi-bar/neuDrive.git` | Upstream — always keep `main` in sync with this |
| `tzj` | `https://github.com/Zijie-Tian/neuDrive.git` | Personal fork — push feature branches here |

**Branches**:
- `main` — tracks `origin/main`, always synced with upstream
- `tzj/neudrive` — personal development branch, contains deployment-specific changes (GOPROXY, Local Mode env)

**Workflow**:
```bash
git checkout main && git pull origin main   # sync upstream
git checkout tzj/neudrive
git merge main                               # merge upstream updates
git push tzj tzj/neudrive                    # push to personal fork
```

## Architecture

### Entry Points (`cmd/`)

- `cmd/neudrive/` — Full CLI with all commands (server, mcp, sync, login, hub operations)
- `cmd/neu/` — Alias for `cmd/neudrive`, same binary
- `cmd/server/` — Standalone HTTP server only
- `cmd/mcp/` — MCP server entry point

### Application Core (`internal/app/`)

- `appcore/appcore.go` — Wires all services, repos, and handlers into a single `App` struct. Supports both SQLite (local dev) and PostgreSQL (production) backends via a `Storage` enum.
- `serverapp/serverapp.go` — HTTP server runner, handles signal shutdown
- `mcpapp/mcpapp.go` — MCP protocol server runner

### HTTP API (`internal/api/`)

`router.go` sets up all routes. Key route groups:
- **Public** (no auth): `/api/config`, `/api/auth/*`, `/login`, `/signup`, OAuth endpoints
- **Local-only** (when `local_mode=true`): `/api/auth/register`, `/api/auth/login` — email/password auth
- **Authenticated**: `/api/*` tree operations, memory, projects, vault, skills
- **Agent/CLI**: `/agent/*` endpoints used by the CLI and local integrations
- **MCP**: `/mcp` — Model Context Protocol SSE endpoint

Auth middleware (`auth.go`) validates JWT Bearer tokens. Local mode auto-generates an owner token via `/api/local/owner-token`.

### Services (`internal/services/`)

Business logic layer. Key services:
- `UserService` / `AuthService` — registration, login, token management
- `FileTreeService` — canonical virtual file tree (the primary data model)
- `MemoryService` — memory entries, scratch notes, conflicts
- `ProjectService` — project logs and structured entries
- `VaultService` — encrypted secrets storage
- `RoleService` — scoped access tokens with trust levels
- `LocalGitSyncService` — Git mirror export/import

### Database

- **Migrations**: `migrations/*.sql` — numbered SQL files, applied automatically on server startup
- **Models**: `internal/models/` — Go structs for all entities
- **Storage backends**: `internal/storage/sqlite/` and `internal/storage/postgres/` (via pgx/v5)

### Frontend (`web/`)

Vite + React SPA. Key structure:
- `src/App.tsx` — router and layout, reads `/api/config` for `local_mode`, `billing_enabled`, etc.
- `src/api.ts` — HTTP client, `API_BASE = "/api"`
- `src/pages/` — page components (LoginPage, SignupPage, DashboardPage, etc.)
- Built output copied to `internal/web/dist/` for Go embed

**Important**: The frontend LoginPage/SignupPage currently only support OAuth providers (GitHub, Pocket ID). They do NOT have email/password forms even when `local_mode=true`. Users in Local Mode must either:
1. Use the API directly (`POST /api/auth/register`, `POST /api/auth/login`) and inject tokens via browser console
2. Or modify the frontend to add Local Mode login forms

### Key Data Model: Canonical File Tree

The core abstraction is a virtual file tree under root paths: `profile/`, `memory/`, `project/`, `skill/`, `secret/`, `platform/`. All user data is stored as nodes in this tree. The API provides read/write/list/search operations against tree paths.

## Environment Variables

See `neudrive.env.example` for full list. Critical ones:
- `PORT` — server listen port
- `DATABASE_URL` — PostgreSQL connection string (production)
- `JWT_SECRET` — token signing key
- `VAULT_MASTER_KEY` — 32-byte hex encryption key for secrets
- `PUBLIC_BASE_URL` — external access URL
- `NEUDRIVE_LOCAL_MODE=1` — enable local email/password auth
- `GITHUB_CLIENT_ID` / `GITHUB_CLIENT_SECRET` — OAuth login
- `NEUDRIVE_ENABLE_BILLING` — show/hide billing UI

## Testing

- Go tests: standard `go test ./...`. Integration tests use in-memory SQLite or a test PostgreSQL DB.
- Frontend e2e: Playwright tests in `web/e2e/`
- CLI integration tests: in `internal/cli/*_integration_test.go`
