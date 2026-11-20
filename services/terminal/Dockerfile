# Rocky Linux 10 Web Terminal for Railway.
#
# Rocky Linux 10 (RHEL 10 compatible) with ttyd serving tmux sessions, behind
# an nginx basic-auth proxy. The shell runs as the non-root user "rocky"
# (passwordless sudo), and /home is a Railway volume. EPEL and CRB are enabled.
# The base image is pinned by dated tag and digest, the other versions below:
# change them and redeploy to upgrade.
FROM rockylinux/rockylinux:10.3.20261112.0@sha256:d24beedc58f79b9e0e9289fedaa4fcbcbe692b1438883a86f177fd7c51053ca8

# Node.js 24 LTS "Krypton" official tarballs. The checksums come from
# https://nodejs.org/dist/v24.22.3/SHASUMS256.txt.
ARG NODE_VERSION=24.22.3
ARG NODE_SHA256_AMD64=4f2b175a6d53a5c617a80dba2676238a0688cef28c6da4eba35c7d844e66fcfb
ARG NODE_SHA256_ARM64=6e51a2ada00f0011cc266f65d98922b403d7ad88f809ca2aa615a5fe1ce6e86c

# tini static release binary (neither Rocky nor EPEL 10 packages tini). The
# checksum comes from https://github.com/krallin/tini/releases/download/v0.19.0/tini-static-amd64.sha256sum.
ARG TINI_VERSION=0.19.0
ARG TINI_SHA256_AMD64=c5b0666b4cb676901f90dfcb37106783c5fe2077b04590973b885950611b30ee
ARG TINI_SHA256_ARM64=eae1d3aa50c48fb23b8cbdf4e369d0910dfc538566bfd09df89a774aa84a48b9

# Extra packages from Rocky, CRB or EPEL to bake into the image, space
# separated, for example "golang postgresql". Railway passes the service
# variable of the same name as a build argument, so changing it rebuilds the
# image with those packages. They survive redeploys, unlike `dnf install` in
# the shell.
ARG ROCKY_EXTRA_PACKAGES=""

ENV LANG=C.UTF-8 \
    TZ=UTC

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

# EPEL (ttyd, ripgrep, fd, htop, ncdu, pipx, fastfetch) needs CRB. Then a
# system upgrade plus the auth proxy (nginx), the web terminal (ttyd 1.7.7
# from EPEL), tmux, a C/C++ toolchain and the tools people expect in a shell.
RUN dnf -y install dnf-plugins-core epel-release \
 && dnf config-manager --set-enabled crb \
 && dnf -y upgrade --refresh \
 && dnf -y install --allowerasing --setopt=install_weak_deps=False \
      bash-completion bind-utils bzip2 ca-certificates curl fastfetch fd-find file \
      findutils gcc gcc-c++ git git-lfs hostname htop iproute iputils jq less libcap \
      lsof make nano ncdu ncurses-term nginx nmap-ncat openssh-clients openssl pipx \
      procps-ng python3 python3-devel python3-pip ripgrep rsync shadow-utils sqlite \
      sudo tar tmux tree ttyd tzdata unzip util-linux vim-enhanced wget which xz zip \
 && dnf clean all \
 && rm -rf /var/cache/dnf \
 && ttyd --version

# Optional extras from ROCKY_EXTRA_PACKAGES, checked as package names first so
# the variable cannot inject shell or dnf options. A value typed into the
# deploy form must not fail the build: invalid names and packages that dnf
# cannot install are skipped with a warning (here and in the deploy log, from
# /etc/rocky-terminal/skipped-packages), and the rest is installed.
RUN set -eux; \
    install -d -m 0755 /etc/rocky-terminal; \
    : > /etc/rocky-terminal/skipped-packages; \
    read -r -a requested <<< "${ROCKY_EXTRA_PACKAGES//,/ }"; \
    extra=(); \
    for pkg in "${requested[@]}"; do \
      if [[ "$pkg" =~ ^[A-Za-z0-9][A-Za-z0-9+._-]*$ ]]; then \
        extra+=("$pkg"); \
      else \
        echo "WARNING: ROCKY_EXTRA_PACKAGES: skipping invalid package name '${pkg:0:64}'" >&2; \
        printf '%s\n' "${pkg:0:64}" >> /etc/rocky-terminal/skipped-packages; \
      fi; \
    done; \
    if (( ${#extra[@]} > 0 )); then \
      if ! dnf -y install --setopt=install_weak_deps=False "${extra[@]}"; then \
        echo "WARNING: ROCKY_EXTRA_PACKAGES: installing '${extra[*]}' together failed. Trying one package at a time." >&2; \
        for pkg in "${extra[@]}"; do \
          if ! dnf -y install --setopt=install_weak_deps=False "$pkg"; then \
            echo "WARNING: ROCKY_EXTRA_PACKAGES: skipping '${pkg}', dnf could not install it." >&2; \
            printf '%s\n' "$pkg" >> /etc/rocky-terminal/skipped-packages; \
          fi; \
        done; \
      fi; \
      dnf clean all; \
      rm -rf /var/cache/dnf; \
    fi; \
    chmod 0644 /etc/rocky-terminal/skipped-packages

# Railway's container capability set lacks NET_RAW, and the kernel refuses to
# exec a binary whose file capabilities exceed that set. Stripping file
# capabilities makes ping fall back to ICMP datagram sockets, which work under
# sudo.
RUN set -eux; \
    caps="$(getcap -r /usr 2> /dev/null || true)"; \
    for bin in $(printf '%s\n' "$caps" | awk 'NF {print $1}'); do setcap -r "$bin"; done; \
    test -z "$(getcap -r /usr 2> /dev/null || true)"

# Node.js LTS from nodejs.org (Rocky 10's default nodejs is 22) and tini.
RUN set -eux; \
    case "$(uname -m)" in \
      x86_64)  narch="x64";   nsum="${NODE_SHA256_AMD64}"; tarch="amd64"; tsum="${TINI_SHA256_AMD64}" ;; \
      aarch64) narch="arm64"; nsum="${NODE_SHA256_ARM64}"; tarch="arm64"; tsum="${TINI_SHA256_ARM64}" ;; \
      *) echo "unsupported architecture: $(uname -m)" >&2; exit 1 ;; \
    esac; \
    tarball="node-v${NODE_VERSION}-linux-${narch}.tar.xz"; \
    curl -fsSL --retry 5 --retry-all-errors --retry-delay 2 -o "/tmp/${tarball}" \
      "https://nodejs.org/dist/v${NODE_VERSION}/${tarball}"; \
    echo "${nsum}  /tmp/${tarball}" | sha256sum -c -; \
    tar -xJf "/tmp/${tarball}" -C /usr/local --strip-components=1 --no-same-owner; \
    rm -f "/tmp/${tarball}" /usr/local/CHANGELOG.md /usr/local/README.md; \
    install -d /usr/local/share/doc/nodejs; \
    mv /usr/local/LICENSE /usr/local/share/doc/nodejs/LICENSE; \
    node --version; \
    npm --version; \
    curl -fsSL --retry 5 --retry-all-errors --retry-delay 2 -o /usr/local/bin/tini \
      "https://github.com/krallin/tini/releases/download/v${TINI_VERSION}/tini-static-${tarch}"; \
    echo "${tsum}  /usr/local/bin/tini" | sha256sum -c -; \
    chmod 0755 /usr/local/bin/tini; \
    /usr/local/bin/tini --version

# The login user "rocky" (uid 1000, bash, passwordless sudo). Its home is
# created on the volume at first boot from /etc/skel: Rocky's default dotfiles
# plus one line in .bashrc that loads this template's shell defaults.
RUN set -eux; \
    printf '\n# Rocky Linux Web Terminal defaults (prompt, history, PATH). Delete these lines to opt out.\nif [ -r /etc/rocky-terminal/bashrc ]; then . /etc/rocky-terminal/bashrc; fi\n' \
      >> /etc/skel/.bashrc; \
    useradd --uid 1000 --user-group --shell /bin/bash --no-create-home --home-dir /home/rocky rocky; \
    rm -rf /home/rocky; \
    printf 'rocky ALL=(ALL:ALL) NOPASSWD: ALL\n' > /etc/sudoers.d/90-rocky-terminal; \
    chmod 0440 /etc/sudoers.d/90-rocky-terminal; \
    visudo -cf /etc/sudoers.d/90-rocky-terminal

COPY entrypoint.sh /usr/local/bin/rocky-terminal-entrypoint
COPY terminal-session.sh /usr/local/bin/terminal-session
COPY bashrc.sh /etc/rocky-terminal/bashrc
COPY motd.txt /etc/rocky-terminal/motd
COPY nginx.conf.template /etc/rocky-terminal/nginx.conf.template
COPY tmux.conf /etc/tmux.conf
RUN chmod 0755 /usr/local/bin/rocky-terminal-entrypoint /usr/local/bin/terminal-session \
 && chmod 0644 /etc/rocky-terminal/bashrc /etc/rocky-terminal/motd \
      /etc/rocky-terminal/nginx.conf.template /etc/tmux.conf

EXPOSE 8080

# On Railway, podman-init is PID 1 and this runs under it. tini -s registers as
# a child subreaper, so orphaned processes (the tmux server, background jobs)
# are reaped whether or not tini is PID 1.
ENTRYPOINT ["/usr/local/bin/tini", "-s", "--", "/usr/local/bin/rocky-terminal-entrypoint"]
