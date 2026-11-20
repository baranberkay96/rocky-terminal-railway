# NOTICE: Rocky Linux 10 Web Terminal template

## Software installed in the image

| Component | Source | License | How it is used |
|---|---|---|---|
| Rocky Linux 10.3 | `rockylinux/rockylinux` image published by the Rocky Enterprise Software Foundation, https://hub.docker.com/r/rockylinux/rockylinux, tag `10.3.20261112.0`, index digest `sha256:d24beedc…` (2026-11-16), upgraded with `dnf upgrade` at build | Various free licenses (see `rpm -qi` and `/usr/share/licenses` in the image) | Base image (`FROM`) and dnf packages |
| EPEL 10 | `epel-release` from Rocky's extras repository, packages from https://dl.fedoraproject.org/pub/epel/10/ | Each package's own license | Repository for ttyd, ripgrep, fd, htop, ncdu, pipx, fastfetch |
| ttyd 1.7.7 | EPEL 10 package `ttyd` | MIT, Copyright (c) 2016-2022 Shuanglei Tao | Web terminal on loopback, installed unmodified |
| nginx 1.26 | Rocky AppStream package `nginx` | BSD-2-Clause | Basic-auth reverse proxy. The config was written for this template |
| tmux 3.3a | Rocky BaseOS package `tmux` | ISC | Persistent sessions |
| tini 0.19.0 | Static release binary from https://github.com/krallin/tini/releases/tag/v0.19.0, SHA-256 checked | MIT, Copyright (c) 2015 Thomas Orozco | Init and child subreaper, unmodified |
| Node.js 24.22.3 LTS | Official tarballs from https://nodejs.org/dist/v24.22.3/, SHA-256 checked | MIT (bundled components under their own licenses, see `/usr/local/share/doc/nodejs/LICENSE`) | Installed unmodified into `/usr/local` |
| GCC, git, Python 3.12 and the other CLI tools | Rocky Linux and EPEL packages | Each package's own license (GPL, MIT, BSD, Apache and similar) | Installed unmodified |

Changes to installed Rocky Linux files: the CRB repository is enabled, file capabilities are removed from all binaries (Railway's containers cannot exec binaries whose capabilities they lack; this lets `sudo ping` work), the account `rocky` (uid 1000, bash) is added, `/etc/sudoers.d/90-rocky-terminal` grants it passwordless sudo, and `/etc/skel/.bashrc` gets one line that loads this template's shell defaults. nginx's own `nginx.conf` is replaced at every start by the file this template renders. No package files are edited otherwise.

Optional packages named in `ROCKY_EXTRA_PACKAGES` are installed from the Rocky Linux, CRB and EPEL repositories at build time under their own licenses.

Product names belong to their owners. This is a community template, not affiliated with the Rocky Enterprise Software Foundation, Red Hat, the Fedora Project (EPEL), the ttyd author, the OpenJS Foundation or Railway. Rocky Linux is a trademark of the Rocky Enterprise Software Foundation; Red Hat Enterprise Linux is a trademark of Red Hat, Inc.

## Upstream files and data checked (2026-11-20)

- Docker Hub tags of `rockylinux/rockylinux`: `10.3.20261112.0` = `10.3` = `10` (index `sha256:d24beedc58f79b9e0e9289fedaa4fcbcbe692b1438883a86f177fd7c51053ca8`, amd64 manifest `sha256:aec7b43a…`, 71 MB compressed).
- Rocky Linux 10 BaseOS, AppStream and CRB `primary.xml` and EPEL 10 `primary.xml` (x86_64): every package name above; `tini` is in none of them, hence the release binary.
- https://nodejs.org/dist/v24.22.3/SHASUMS256.txt and https://github.com/krallin/tini/releases/tag/v0.19.0 (`tini-static-amd64`/`arm64` `.sha256sum`).
- The ttyd behaviour this template relies on (auth-header mode `-H`, loopback `-i lo`, the `-c` startup log line that prints the credential) was checked for the Debian Web Terminal template of the same author against ttyd tag `1.7.7`; EPEL ships the same release.

## Market templates reviewed

"Rocky Linux" (bare `rockylinux:9` with `tail -f /dev/null` and a `/data` volume) and "Rocky Linux Terminal" (repo `Amritasha/rocky-railway`, Rocky Linux 9 with ttyd, volume at `/root`). Only their configuration and listing figures were used; no files or text from other templates were used.

All files in this package were written for this template, adapted from the same author's Ubuntu, Debian and Kali Linux Web Terminal templates: the Dockerfile, `entrypoint.sh`, `terminal-session.sh`, `bashrc.sh`, `tmux.conf`, `nginx.conf.template`, `motd.txt` and the docs.
