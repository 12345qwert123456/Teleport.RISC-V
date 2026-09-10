# Teleport RISC-V 64 Build

Automated cross-compilation of [Gravitational Teleport](https://github.com/gravitational/teleport) for **linux/riscv64**.

Teleport is an infrastructure access platform for servers, Kubernetes clusters, databases, and applications. Official riscv64 support does not exist — this repository fills that gap.

New Teleport releases are detected daily and built automatically.

## Binaries

The full set of Teleport's `linux` binaries is built (same list as upstream `BINS_default`):

- `teleport` — main daemon (server and agent)
- `tsh` — CLI client
- `tctl` — administration tool
- `tbot` — Machine ID / Workload Identity agent
- `teleport-update` — self-update client for managed agent upgrades
- `fdpass-teleport` — file-descriptor passing helper for `tsh` connection resumption (Rust)

## Quick Start

### Build locally

```bash
# Default version (TELEPORT_VERSION in the Dockerfile)
docker buildx build --platform linux/riscv64 -t teleport:riscv64 -f Dockerfile.teleport .

# Specific version
docker buildx build --platform linux/riscv64 --build-arg TELEPORT_VERSION=v18.10.0 \
  -t teleport:18.10.0-riscv64 -f Dockerfile.teleport .
```

> Builder stages run natively on the build host (no QEMU). The resulting image is
> `linux/riscv64` and contains riscv64 binaries cross-compiled with `gcc-riscv64-linux-gnu`.

### Download binaries

Pre-built binaries are published as GitHub Releases automatically.
Go to the [Releases](../../releases) page, download the `.tar.gz` for your version:

```bash
tar xzf teleport-*-linux-riscv64-bin.tar.gz
sudo install -m 755 teleport tsh tctl tbot teleport-update fdpass-teleport /usr/local/bin/
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

This makes `CGO_ENABLED=0` impossible for `teleport`, `tsh` and `tctl`. Instead,
cross-compilation is used with `gcc-riscv64-linux-gnu` on an amd64 host — with no
patches to the Teleport source code. `tbot` and `teleport-update` build as pure Go
(`CGO_ENABLED=0`). `fdpass-teleport` is a small Rust program, cross-compiled with
the `riscv64gc-unknown-linux-gnu` Rust target (linker: `riscv64-linux-gnu-gcc`).

### Build stages

| Stage | Base image | Purpose |
|-------|------------|---------|
| `source` | `golang:1.25-trixie` (build host) | Clones Teleport once, resolves toolchain versions from it |
| `webassets_builder` | `node:24-alpine` (build host) | Builds the web UI (pnpm + Rust/WASM) |
| `go_builder` | `golang:1.25-trixie` (build host) | Cross-compiles the Go binaries + Rust `fdpass-teleport` for riscv64 |
| final | `ubuntu:24.04` (riscv64) | Minimal runtime image (COPY only) |

Toolchain versions are read from the Teleport source being built, so they follow upstream
automatically: Rust from `rust-toolchain.toml` (or `build.assets/versions.mk` on older tags),
`wasm-bindgen-cli` from `Cargo.lock`, `wasm-opt` from `versions.mk`, pnpm from
`package.json` `packageManager`.

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
- Builds the `linux/riscv64` image with Docker BuildKit (no QEMU)
- Verifies every extracted binary is a genuine RISC-V ELF
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
| Go | 1.25 (`golang:1.25-trixie`, `GOTOOLCHAIN=auto` follows `go.mod`) |
| Rust | from upstream (target `riscv64gc-unknown-linux-gnu`) |
| GCC cross | 14.2.0 (riscv64-linux-gnu) |
| Node.js | 24 |
| pnpm | from upstream `packageManager` |
| wasm-bindgen / wasm-opt | from upstream `Cargo.lock` / `versions.mk` |

## License

Teleport is licensed under [GNU AGPL v3](https://github.com/gravitational/teleport/blob/master/LICENSE).
Build files in this repository are MIT licensed.
