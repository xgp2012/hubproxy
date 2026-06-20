# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

HubProxy is a lightweight, high-performance multi-functional proxy service written in Go. It provides:
- Docker image registry acceleration (Docker Hub, GHCR, Quay, registry.k8s.io)
- GitHub file acceleration (Releases, Raw files, API endpoints)
- Hugging Face model download acceleration
- Offline Docker image package download with debouncing
- Docker image search
- IP-based rate limiting with whitelist/blacklist
- Custom access control (whitelist/blacklist) for Docker images and GitHub repos
- Token/Manifest caching for performance
- HTTP/2 cleartext (H2C) support
- Embedded frontend static pages

## Codebase Structure

```
src/
├── main.go                  # Entry point, server setup, route registration
├── config.toml              # Default configuration (embedded into binary)
├── config/
│   └── config.go            # AppConfig struct, LoadConfig(), env var overrides
├── handlers/
│   ├── docker.go            # Docker Registry API v2 proxy (manifests, blobs, tags, auth)
│   ├── github.go            # GitHub proxy handler, URL matching via regex, script processing
│   ├── imagetar.go          # Offline image stream download (single & batch), debouncer, token store
│   └── search.go            # Docker Hub image search with pagination and caching
├── utils/
│   ├── access_control.go    # AccessController for Docker image & GitHub repo whitelist/blacklist
│   ├── cache.go             # UniversalCache for token/manifest caching, TTL management
│   ├── http_client.go       # Global and search-specific HTTP clients
│   ├── proxy_shell.go       # Smart shell script URL rewriting (.sh, .ps1)
│   └── ratelimiter.go       # IPRateLimiter with per-IP rate limiting, cleanup routine
└── public/                  # Embedded frontend: index.html, images.html, search.html, favicon.ico
```

## Key Architecture Details

- **Router**: Built with Gin. Routes are registered in `buildRouter()` in `main.go`. Docker proxy routes (`/token`, `/v2`) and GitHub proxy (via `NoRoute`) are handled separately from frontend pages.
- **Docker Proxy**: Uses `github.com/google/go-containerregistry` for registry interactions. Supports multi-registry backends via `registries` config section. Auth tokens and manifests are cached in `utils/cache.go`.
- **GitHub Proxy**: The `NoRoute` handler in `main.go` catches all unmatched paths, parses them as GitHub URLs via regex patterns, and proxies the requests. Shell scripts (.sh, .ps1) get special URL rewriting to nested-accelerate GitHub links.
- **Image Download**: `imagetar.go` streams Docker image layers into tar/tar.gz format with debouncer-based anti-spam and token-based download authorization.
- **Config**: Loads from `config.toml` (default config path) with environment variable overrides taking precedence. Config is cached for 5 seconds to reduce lock contention.

## Development Commands

```bash
# Build
cd src && go build -o hubproxy .

# Run tests
cd src && go test ./...

# Run with specific config
CONFIG_PATH=/path/to/config.toml ./hubproxy

# Run with environment variable overrides
SERVER_PORT=8080 ./hubproxy
```

## Building

```bash
# Local binary build
cd src && CGO_ENABLED=0 go build -ldflags="-s -w -X main.Version=v1.0.0" -trimpath -o hubproxy .

# Docker build (multi-platform)
docker buildx build --push --platform linux/amd64,linux/arm64 --tag ghcr.io/sky22333/hubproxy:latest -f Dockerfile .
```

## CI/CD Workflows

- `.github/workflows/docker-ghcr.yml` — Builds and pushes multi-platform Docker image to GHCR (triggered by `workflow_dispatch` with version input)
- `.github/workflows/release.yml` — Builds binaries for linux-amd64/linux-arm64, creates .deb/.rpm/.apk packages via nFPM, and publishes GitHub Releases (triggered by `workflow_dispatch`)

## Configuration

Configuration is managed through `src/config.toml` and can be overridden via environment variables. Key sections:
- `[server]` — host, port, file size limit, H2C, frontend toggle
- `[rateLimit]` — per-IP request limit and period
- `[security]` — IP whitelist/blacklist (CIDR supported)
- `[access]` — Docker/Repo whitelist/blacklist with wildcard support, proxy settings
- `[registries]` — upstream registry mappings (GHCR, GCR, Quay, K8s)
- `[tokenCache]` — token/manifest cache enabled flag and default TTL

The config package (`config/config.go`) provides `GetConfig()` which returns a thread-safe copy with 5-second cache TTL.

## Runtime

Default listen address: `0.0.0.0:5000`. Health check endpoint at `/ready`. The service embeds static frontend files via `//go:embed public/*`. Frontend can be disabled via `enableFrontend = false`.
