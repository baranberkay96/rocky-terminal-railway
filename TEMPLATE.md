# Rocky Linux 10 Web Terminal (Browser Shell): Railway template spec

## Listing

| Field | Value |
|---|---|
| Name | **Rocky Linux 10 Web Terminal (Browser Shell)** ("10" is the major release; the image is 10.3) |
| Description | RHEL-compatible Rocky Linux 10 shell in your browser, password-protected. (73 chars) |
| Category | Other |
| Tags | rocky-linux, terminal, web-terminal, linux, shell (the first five are sent), rhel, ttyd, tmux, rocky-10 |
| Icon | `https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/svg/rocky-linux.svg` (200, image/svg+xml) |

Why: the Rocky niche had about 17 deploys in 30 days (marketplace data, 2026-11). "Rocky Linux" runs bare `rockylinux:9` with `tail -f /dev/null`; "Rocky Linux Terminal" (Amritasha) runs Rocky 9 with ttyd and a `/root` volume. This package moves to Rocky Linux 10 and brings our web terminal design (116, 60, 109, 127).

## Base

Derived from `116-ubuntu-terminal` and `127-alpine-terminal` (same nginx auth proxy, `/healthz`, ttyd auth-header mode, tmux sessions, fail-closed password handling per STANDARD 8.29, build-time extra packages like 109's `KALI_EXTRA_PACKAGES`).

Rocky-specific changes:
- Base `rockylinux/rockylinux`, the Rocky Enterprise Software Foundation's own Docker Hub namespace (the Docker official `rockylinux` image carries the same 10.3 release).
- `epel-release` + CRB enabled first: ttyd 1.7.7, ripgrep, fd, htop, ncdu, pipx and fastfetch come from EPEL 10. Then `dnf upgrade` and one `dnf install --allowerasing --setopt=install_weak_deps=False`.
- tini from the static GitHub release with SHA-256 (no Rocky/EPEL package); Node.js 24 LTS from the nodejs.org tarball with SHA-256 (Rocky 10's default `nodejs` is 22).
- tmux is 3.3a (Rocky BaseOS); the shared `tmux.conf` only uses options that exist in 3.3.
- nginx runs as `nginx`; temporary paths in `/run/terminal-nginx`; nginx starts with `-e stderr`.
- Rocky Linux 10 requires x86-64-v3 CPUs; Railway's hosts provide it.

## Pinned versions (checked 2026-11-20)

| Component | Source | Version |
|---|---|---|
| Base image | `rockylinux/rockylinux:10.3.20261112.0@sha256:d24beedc58f79b9e0e9289fedaa4fcbcbe692b1438883a86f177fd7c51053ca8` | Rocky Linux 10.3 |
| ttyd | EPEL 10 | 1.7.7 |
| Node.js | nodejs.org tarball, SHA-256 checked | 24.22.3 LTS |
| nginx | Rocky AppStream | 1.26.3 |
| tmux | Rocky BaseOS | 3.3a |
| Python | Rocky BaseOS | 3.12 |
| tini | GitHub release, SHA-256 checked | 0.19.0 |

Every build runs `dnf upgrade`, so Rocky 10 and EPEL updates are picked up.

## Resource estimate

Railway prices: RAM $10/GB-month, vCPU $20/vCPU-month, volume $0.15/GB-month.

| Service | Idle | Typical | Notes |
|---|---|---|---|
| Terminal | ~0.03 GB RAM (estimate from 127/128/130, not yet measured here) | 0.3 GB avg | Peaks come from what users run |
| Volume `/home` | 0.1 GB | 2-5 GB | Projects, venvs, node_modules |

Estimated monthly cost: idle about $0.40, typical about $4. Minimum plan: Free. Image about 0.4 GB compressed (estimate: 304 RPMs, 235 MB of RPMs, about 0.95 GB installed, plus Node.js), below the 4 GB cap.

## Services

### Terminal

| Setting | Value |
|---|---|
| Source | repo `baranberkay96/rocky-terminal-railway`, root `/services/terminal` (Dockerfile) |
| Healthcheck | `/healthz` on `PORT=8080`, timeout 120 s |
| Public domain | yes, target port 8080 |
| Volume | `/home` (home of `rocky` is `/home/rocky`) |
| Restart policy | ON_FAILURE, 10 retries |
| Replicas | 1 |

### Variables

| Variable | Default | Notes |
|---|---|---|
| `PASSWORD` | `${{secret(32)}}` | 12-256 bytes, one line. Otherwise the login is locked with a random password; the deploy still succeeds and the log says how to fix it. |
| `USERNAME` | `rocky` | Invalid names fall back to `rocky` with a warning. |
| `PORT` | `8080` | Invalid values or 7681 fall back to 8080 with a warning. |
| `TERMINAL_TMUX` | `true` | `false`: plain login shell per tab. |
| `TERMINAL_FONT_SIZE` | `15` | 8-40; anything else falls back to 15 with a warning. |
| `ROCKY_EXTRA_PACKAGES` | empty | Build argument. Packages from Rocky, CRB or EPEL. Invalid names and packages dnf cannot install are skipped with a warning (build log and deploy log); never fails the build. |
| `TZ` | `UTC` | Shell time zone. |
| `RAILWAY_DEPLOYMENT_DRAINING_SECONDS` | `15` | Grace period for the clean tmux shutdown. |

No required inputs.

## Resource estimate

Railway prices: RAM $10/GB-month, vCPU $20/vCPU-month, volume $0.15/GB-month.

RESOURCE_TABLE

## Improvements over market templates

1. Rocky Linux 10 instead of 9.
2. Generated `secret(32)` password, nginx basic auth with a SHA-512 crypt hash, rate limit, cross-site WebSocket check, loopback-only ttyd; the password never reaches logs (ttyd `-c` prints it). Fails closed without failing the deploy.
3. The whole `/home` persists (dotfiles, history, venvs, pipx, `npm -g`) for a non-root user with sudo. Market: a root shell with `/root` only, or `/data` only.
4. tmux sessions survive closed tabs; `?arg=NAME` opens separate sessions.
5. EPEL and CRB enabled, a C/C++ toolchain and the usual CLI tools.
6. Pinned base image (dated tag + digest); `dnf upgrade` at build; Node.js 24 LTS with checksum.
7. `/healthz` healthcheck with `PORT` set; build-time extra packages that never fail the build.

## Verification checklist

- [x] Pre-publish test deploy (2026-11-20): build + deploy SUCCESS first try, `/healthz` 200.
- [ ] Hands-on checks not run yet (workspace billing pause): login 401/200, WebSocket + foreign Origin, `whoami`/`sudo`/node/python/gcc/`sudo ping`, venv + npm, persistence after redeploy, `PASSWORD=abc` lock, RAM under a 0.5 GB limit.
- [ ] Publish + post-publish test.

## Open questions

- EPEL 10 follows the newest RHEL 10 minor release; a package that EPEL rebuilds against a newer minor than the base can wait a few days for Rocky to catch up.
