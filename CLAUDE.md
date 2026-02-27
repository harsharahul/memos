# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Memos is a self-hosted, open-source note-taking/knowledge management service. Go backend (single binary) + React frontend, with SQLite/MySQL/PostgreSQL support. See AGENTS.md for detailed architecture documentation.

**This is a fork** of [usememos/memos](https://github.com/usememos/memos). See the Fork Management section below for practices around syncing and custom work.

## Development Commands

### Backend (Go 1.25)

```bash
go run ./cmd/memos --port 8081          # Start dev server
go test ./...                            # Run all tests
go test ./store/...                      # Test specific package
go test ./server/router/api/v1/test/...  # Test API layer
golangci-lint run                        # Lint
```

Test against other databases:
```bash
DRIVER=mysql DSN="user:pass@tcp(localhost:3306)/memos" go test ./...
DRIVER=postgres DSN="postgres://user:pass@localhost:5432/memos" go test ./...
```

### Frontend (React 18 + TypeScript + Vite 7)

```bash
cd web
pnpm install        # Install dependencies
pnpm dev            # Dev server on :3001 (proxies API to :8081)
pnpm lint           # TypeScript check + Biome lint
pnpm lint:fix       # Auto-fix lint issues
pnpm build          # Production build
pnpm release        # Build + copy to server/router/frontend/dist
```

### Protocol Buffers

```bash
cd proto && buf generate                  # Regenerate Go + TypeScript
cd proto && buf lint                      # Lint proto files
cd proto && buf breaking --against .git#main  # Breaking change check
```

## Architecture

### Backend Layers

```
cmd/memos/main.go → server/server.go (Echo HTTP)
  ├── server/router/api/v1/*_service.go   # Service implementations
  ├── server/router/frontend/             # SPA serving
  ├── server/router/fileserver/           # Media file serving
  └── server/runner/                      # Background jobs
        ↓
  store/store.go (caching wrapper) → store/driver.go (DB interface)
        ↓
  store/db/{sqlite,mysql,postgres}/       # Three DB implementations
```

**Dual API protocol:** Connect RPC for browser clients (`/memos.api.v1.*`) and gRPC-Gateway for REST (`/api/v1/*`). Both use the same service implementations.

**Auth:** JWT (15-min, stateless) + Personal Access Tokens (stateful, SHA-256 hashed). See `server/auth/authenticator.go`.

**Public endpoints** must be listed in `server/router/api/v1/acl_config.go`.

### Frontend State

- **Server state:** React Query v5 via hooks in `web/src/hooks/` (e.g., `useMemoQueries.ts`)
- **Client state:** React Context (`AuthContext`, `ViewContext`, `MemoFilterContext`)
- **API client:** Connect RPC (`web/src/lib/connect.ts`)
- **Styling:** Tailwind CSS v4

### Database Migrations

Migration files live in `store/migration/{sqlite,mysql,postgres}/{version}/`. When changing the schema:
1. Create migration SQL for all three drivers
2. Update `store/migration/{driver}/LATEST.sql`
3. Implement in `store/driver.go` interface + all three DB implementations

## Code Conventions

### Go
- Use `errors.Wrap(err, "context")` from `github.com/pkg/errors` (not `fmt.Errorf`)
- Return gRPC status errors: `status.Errorf(codes.NotFound, "message")`
- Imports: stdlib, third-party, local (`github.com/usememos/memos`) — use `goimports`
- Forbidden by linter: `fmt.Errorf`, `ioutil.ReadDir`

### TypeScript/React
- Biome for linting/formatting (line width 140, semicolons always, 2-space indent)
- Absolute imports with `@/` alias
- Generated proto types in `web/src/types/proto/` — do not edit manually

## Key Workflows

### Adding an API Endpoint
1. Define in `proto/api/v1/*_service.proto`
2. Run `cd proto && buf generate`
3. Implement in `server/router/api/v1/*_service.go`
4. If public: add to `acl_config.go`
5. Add frontend hook in `web/src/hooks/`

### Adding a Plugin
Plugins live in `plugin/` (scheduler, email, filter, webhook, markdown, httpgetter, storage/s3).

## CI Checks

- **Backend:** `go mod tidy` check, golangci-lint, `go test ./...` (triggered by Go file changes)
- **Frontend:** `pnpm lint`, `pnpm build` (triggered by `web/**` changes)
- **Proto:** `buf lint`, `buf breaking` (triggered by `.proto` changes)
- **Sync upstream:** Weekly auto-sync of `main` from upstream (`.github/workflows/sync-upstream.yml`), also triggerable manually

## Fork Management

### Remotes

- `origin` — `harsharahul/memos` (this fork)
- `upstream` — `usememos/memos` (source project)

### Branch Strategy

- **`main`** tracks `upstream/main` and may contain fork-specific tooling (CLAUDE.md, sync workflow). Feature work still goes on `harsha/*` branches.
- **Custom work** goes on `harsha/`-prefixed feature branches (e.g., `harsha/distributedsession`).
- Automated weekly sync via `.github/workflows/sync-upstream.yml` merges upstream into `main`. Can also be triggered manually from the Actions tab.

### Active Fork Branches

| Branch | Purpose |
|--------|---------|
| `harsha/distributedsession` | Redis session management for horizontal scaling in K8s |

### Manual Sync & Rebase

```bash
# Sync main from upstream
git fetch upstream
git checkout main
git merge upstream/main --no-edit
git push origin main

# Rebase a feature branch onto updated main
git checkout harsha/distributedsession
git rebase main
git push origin harsha/distributedsession --force-with-lease
```

### Rules for Fork Work

1. **Keep `main` minimal** — only fork-specific tooling (CLAUDE.md, workflows) on `main`. Feature work goes on `harsha/*` branches.
2. **Rebase, don't merge** upstream into feature branches — keeps history linear and conflicts localized.
3. **Check for overlap before syncing** — run `git diff main..upstream/main --stat` and compare with files your branches touch.
4. **Update the branch table above** when adding or removing custom branches.
5. **Contribute general-purpose changes upstream** via PR to `usememos/memos` when possible — reduces long-term maintenance burden.
