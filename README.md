# Deploy and Host Rocky Linux 10 Web Terminal with Railway

[![Deploy on Railway](https://railway.com/button.svg)](https://railway.com/deploy/rocky-linux-10-web-terminal-browser-shel?referralCode=20H68q)

A Rocky Linux 10 shell, binary-compatible with Red Hat Enterprise Linux 10, that opens in any browser. You work as the user `rocky` with sudo, inside tmux, so sessions and long-running jobs survive closed tabs. A generated password protects the login, and your home directory sits on a persistent volume. Git, Python, Node.js 24, GCC and everyday CLI tools come preinstalled.

## About Hosting Rocky Linux 10 Web Terminal

A bare `rockylinux` container on Railway has no way in except Railway's own CLI. This community template turns it into a working terminal: ttyd serves the shell over HTTPS, nginx stands in front of it and checks a password that Railway generates at deploy, and tmux keeps your work running between visits. The image builds on the Rocky Enterprise Software Foundation's `rockylinux/rockylinux` image from Docker Hub, pinned by dated tag and digest, enables EPEL and CRB, and runs `dnf upgrade` on every build. `/home` is a Railway volume, Railway's edge provides HTTPS, and the container restarts automatically if it stops. The password never shows up in logs.

## Common Use Cases

- A RHEL-compatible shell for testing scripts, builds and RPM packages before they reach Red Hat Enterprise Linux 10 servers
- A cloud Linux shell for Chromebooks, tablets, locked-down laptops or any machine without SSH
- Running long jobs and scripts in tmux and checking back on them later
- Learning RHEL-style administration (dnf, EPEL, RPM) on a disposable machine without a subscription
- A small always-on workspace for Git, Python, Node.js and C/C++ projects

## Dependencies for Rocky Linux 10 Web Terminal Hosting

- A Railway volume mounted at `/home` (included in the template)
- Nothing else: no database, no API keys, no external accounts

### Deployment Dependencies

- Rocky Linux documentation: https://docs.rockylinux.org/
- Rocky Linux image on Docker Hub: https://hub.docker.com/r/rockylinux/rockylinux
- EPEL: https://docs.fedoraproject.org/en-US/epel/
- ttyd (web terminal, MIT): https://github.com/tsl0922/ttyd
- tmux: https://github.com/tmux/tmux/wiki
- Node.js releases: https://nodejs.org/en/about/previous-releases
- Railway build variables: https://docs.railway.com/builds/dockerfiles#using-variables-at-build-time
- Railway Fair Use Policy: https://railway.com/legal/fair-use

### Implementation Details

| Service | Source | Purpose |
|---|---|---|
| Terminal | Dockerfile based on `rockylinux/rockylinux:10.3.20261112.0` | nginx basic-auth proxy on port 8080 (public HTTPS domain, `/healthz`), ttyd on loopback, tmux sessions, user `rocky` with sudo, volume at `/home` |

**Plan:** works on every plan, including Free (0.5 GB RAM per service). Measured on Railway with a 0.5 GB memory limit: about 0.027 GB of RAM idle, and about 0.06 GB while a Python venv and an npm package were being installed. The image is about 0.4 GB compressed. Large builds can need more than 0.5 GB; on Free the kernel then stops that process, not the terminal.

**First login**

1. Open the service's public URL.
2. Your browser asks for a user name and password. The user name is `rocky` (the `USERNAME` variable). The password is the generated `PASSWORD` value on the service's Variables tab.
3. You land in a tmux session named `main`. Close the tab at any time and reopen the URL to resume. For a second, independent session, open `https://{your-domain}/?arg=work`.
4. `sudo -i` gives you a root shell.

**Tools included**: git, git-lfs, curl, wget, vim, nano, tmux, htop, ncdu, tree, jq, ripgrep, fd, rsync, sqlite3, zip/unzip, dig, ping, ncat, openssh-clients, GCC and G++ with make, Python 3.12 with pip, pipx and headers, and Node.js 24 LTS with npm (from nodejs.org, checksum-verified). `npm install -g` installs into `~/.npm-global`, so it works without sudo and persists. Use `python3 -m venv` or `pipx` for Python packages.

**More packages**: set the `ROCKY_EXTRA_PACKAGES` variable to a space-separated list of packages from Rocky Linux, CRB or EPEL, for example `golang postgresql`. Railway passes the variable to the Docker build, so the next deploy builds an image with those packages and they survive every redeploy. Names dnf cannot install are skipped with a warning in the build and deploy logs, so a typo never fails the deploy. Packages you install with `sudo dnf install` in the shell work right away but reset on the next redeploy.

**What persists**: everything under `/home/rocky`: files, Git repositories, shell history, dotfiles, Python virtualenvs and pipx apps in `~/.local`, global npm packages in `~/.npm-global`. Running processes and packages installed with dnf reset on every redeploy or restart. tmux keeps sessions alive across browser reconnects, not across container restarts.

**Security**: the shell is never served without a password. With an empty, short (under 12 characters) or overlong `PASSWORD` the service still starts, but the login is locked with a random password nobody knows and the deploy log says how to fix it. Invalid `PORT`, `USERNAME` or `TERMINAL_FONT_SIZE` values fall back to the defaults with a warning in the log. nginx checks the password, which the container keeps only as a SHA-512 crypt hash, and rate-limits attempts. It also rejects WebSocket connections from other sites. ttyd listens on 127.0.0.1 only. The password does not appear in logs, process arguments or the shell environment. To change it, edit `PASSWORD` and redeploy. Browsers cache basic-auth logins until you close them, so use a private window on shared computers.

**Limits**: this is a container, not a VM. There is no systemd, no Docker inside the shell, no raw sockets (`ping` works with `sudo`), and only HTTP(S) on the public domain. Railway's [Fair Use Policy](https://railway.com/legal/fair-use) applies to everything that runs in this shell.

**Scaling**: this is a single-user machine on one volume, so keep one replica. Railway scales RAM and CPU vertically up to your plan limit. For a team, deploy one terminal per person.

**Pinning and upgrades**: the Rocky Linux base image is pinned by dated tag and digest on the `FROM` line of `services/terminal/Dockerfile`, and Node.js by version and SHA-256 checksum. Every fresh build runs `dnf upgrade`, so updates released after the base image are included. Rocky Linux 10 is supported until 2035; minor releases (10.3, 10.4, …) arrive through `dnf upgrade`, or change the tag and digest and redeploy.

### Why Deploy Rocky Linux 10 Web Terminal on Railway?

Railway gives the terminal an HTTPS URL, a persistent volume and automatic restarts, with no server to administer. You get a protected, RHEL-compatible Rocky Linux 10 shell from one click and pay only for the CPU and memory it uses. This is a community template, not affiliated with the Rocky Enterprise Software Foundation, Red Hat, the ttyd author or Railway. Rocky Linux is a trademark of the Rocky Enterprise Software Foundation; Red Hat Enterprise Linux is a trademark of Red Hat, Inc.
