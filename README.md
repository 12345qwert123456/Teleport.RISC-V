# Teleport RISC-V 64 Build

Automated cross-compilation of [Gravitational Teleport](https://github.com/gravitational/teleport) for **linux/riscv64**.

Teleport is an infrastructure access platform for servers, Kubernetes clusters, databases, and applications. Official riscv64 support does not exist — this repository fills that gap.

New Teleport releases are detected daily and built automatically.

## Binaries

- `teleport` — main daemon (server and agent)
- `tsh` — CLI client
- `tctl` — administration tool

## Quick Start

### Build locally

```bash
# Latest version
docker buildx build -t teleport:riscv64 -f Dockerfile.teleport .

# Specific version
docker buildx build --build-arg TELEPORT_VERSION=v18.7.2 \
  -t teleport:18.7.2-riscv64 -f Dockerfile.teleport .
```

> The build runs on an amd64 host. The image contains riscv64 binaries
> cross-compiled with `gcc-riscv64-linux-gnu`.

### Download binaries

Pre-built binaries are published as GitHub Releases automatically.
Go to the [Releases](../../releases) page, download the `.tar.gz` for your version:

```bash
tar xzf teleport-*-linux-riscv64-bin.tar.gz
sudo install -m 755 teleport tsh tctl /usr/local/bin/
```

### Docker image

```bash
# Pull from Docker Hub
docker pull 12345qwert123456/teleport-riscv64:latest

# Run on a RISC-V device
docker run -d --name teleport \
  -v /etc/teleport:/etc/teleport \
  -v /var/lib/teleport:/var/lib/teleport \
  -p 3025:3025 -p 3080:3080 -p 3022:3022 \
  12345qwert123456/teleport-riscv64:latest start -c /etc/teleport/teleport.yaml
```

## How It Works

Teleport requires CGO for two packages:
- `lib/system` — signal handling (`signal.h`)
- `lib/shell` — user login shell lookup (`getpwnam_r` from `pwd.h`)

This makes `CGO_ENABLED=0` impossible. Instead, cross-compilation is used with
`gcc-riscv64-linux-gnu` on an amd64 host — with no patches to the Teleport source code.

### Build stages

| Stage | Base image | Purpose |
|-------|------------|---------|
| `webassets_builder` | `node:24-alpine` (amd64) | Builds the web UI (pnpm + Rust/WASM) |
| `go_builder` | `golang:1.25` (amd64) | Cross-compiles Go binaries for riscv64 |
| final | `ubuntu:24.04` | Minimal runtime image |

### Key environment variables

```
CC=riscv64-linux-gnu-gcc    # cross-compiler
CGO_ENABLED=1               # required by Teleport
GOOS=linux GOARCH=riscv64   # target architecture
```

## CI/CD

The GitHub Actions workflow (`.github/workflows/build-riscv64.yml`):
- Runs daily at 06:00 UTC (and on manual trigger)
- Fetches the latest Teleport release version from the GitHub API
- Checks if a riscv64 release already exists — skips if it does
- Builds the image with Docker BuildKit and GHA layer cache
- Verifies the binary is a genuine RISC-V ELF
- Publishes standalone binaries (`.tar.gz` + SHA256) as a GitHub Release
- Pushes the Docker image to Docker Hub

Force a rebuild or build a specific version via `workflow_dispatch`.

### Required repository settings

| Type | Name | Description |
|------|------|-------------|
| Variable | `DOCKERHUB_USERNAME` | Docker Hub username |
| Secret | `DOCKERHUB_TOKEN` | Docker Hub access token |

## Requirements

- Docker with BuildKit (Docker Desktop or `docker buildx`)
- ~16 GB RAM
- ~30 minutes for a full build without cache

## Versions

| Component | Version |
|-----------|---------|
| Teleport | latest (auto-detected) |
| Go | 1.25 |
| GCC cross | 14.2.0 (riscv64-linux-gnu) |
| Node.js | 24 |
| wasm-bindgen | 0.2.99 |

## License

Teleport is licensed under [GNU AGPL v3](https://github.com/gravitational/teleport/blob/master/LICENSE).
Build files in this repository are MIT licensed.
