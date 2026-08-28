# code-server on Coolify — customised image on top of the published release.
#
# Use this when you want extra system packages, language runtimes, or
# pre-installed extensions, but you have NOT modified code-server's own source.
# It builds in a couple of minutes instead of the hour-plus a source build takes.
#
# In Coolify: build pack "Dockerfile", Dockerfile location
# `/coolify/Dockerfile`, base directory `/`, Ports Exposes `8080`.
# See docs/coolify.md.
#
# This file deliberately COPYs nothing. The repository root .dockerignore
# excludes everything except ci/ and release-packages/, so a COPY from any
# other path would fail; keeping the build context unused avoids that entirely.

ARG CODE_SERVER_VERSION=4.135.0
FROM codercom/code-server:${CODE_SERVER_VERSION}

USER root

# Anything your projects need at runtime. Trim this list to taste.
RUN apt-get update \
  && apt-get install -y --no-install-recommends \
    build-essential \
    jq \
    python3 \
    python3-pip \
    python3-venv \
    ripgrep \
    unzip \
  && rm -rf /var/lib/apt/lists/*

# Extra code-server flags without fighting the image's ENTRYPOINT: it already
# ends with a positional workspace argument, so appending a Compose `command:`
# would pass a second one. A config file avoids that. Command-line flags still
# win over this file, and $PASSWORD still wins over a `password` key here.
RUN mkdir -p /etc/code-server \
  && printf '%s\n' \
    'auth: password' \
    'disable-telemetry: true' \
    'disable-update-check: true' \
    > /etc/code-server/config.yaml \
  && chown -R coder:coder /etc/code-server
ENV CODE_SERVER_CONFIG=/etc/code-server/config.yaml

# Extensions go into /opt, not into /home/coder, because a volume mounted at
# /home/coder shadows whatever the image put there on every deploy after the
# first. A startup script seeds them into the home directory once, so they are
# present on a fresh volume and afterwards the user's own set is what persists.
RUN mkdir -p /opt/code-server/extensions /opt/code-server/entrypoint.d \
  && printf '%s\n' \
    '#!/bin/sh' \
    'set -eu' \
    'src=/opt/code-server/extensions' \
    'dest="$HOME/.local/share/code-server/extensions"' \
    'if [ -d "$src" ] && [ ! -d "$dest" ]; then' \
    '  mkdir -p "$(dirname "$dest")"' \
    '  cp -a "$src" "$dest"' \
    '  echo "Seeded baked extensions into $dest"' \
    'fi' \
    > /opt/code-server/entrypoint.d/00-seed-extensions.sh \
  && chmod +x /opt/code-server/entrypoint.d/00-seed-extensions.sh \
  && chown -R coder:coder /opt/code-server

# The published image sets ENTRYPOINTD from an unset ${HOME} at build time, so
# point it somewhere the home volume cannot shadow.
ENV ENTRYPOINTD=/opt/code-server/entrypoint.d

USER 1000

# Copying extensions.json along with the folders is what makes them register,
# so install into the seed directory rather than assembling it by hand.
RUN code-server \
  --extensions-dir /opt/code-server/extensions \
  --install-extension ms-python.python \
  --install-extension esbenp.prettier-vscode
