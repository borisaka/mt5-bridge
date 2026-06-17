FROM debian:12

# Prevent interactive prompts during package installation and set default environment variables
ENV DEBIAN_FRONTEND=noninteractive
ENV USER=root

# Update and install necessary packages (excluding xfce4-power-manager)
RUN apt-get update && apt-get install -y \
    xfce4 \
    xfce4-goodies \
    xorg \
    tightvncserver \
    novnc \
    websockify \
    dbus-x11 \
    wget \
    curl \
    unzip \
    && apt-get remove -y xfce4-power-manager \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# --- Wine, baked at build time. DEFAULT = a PORTABLE Kron4ek build (no apt deps) ---
# The winehq apt packages are a dead end on this host: stable bootstraps but the
# MT5 installer's VMProtect anti-debug blocks; 10.x's classic i386 WoW64 won't
# bootstrap ("could not load kernel32 c0000135") and amd64-only can't be installed
# (wine-staging hard-depends on wine-staging-i386). A portable Kron4ek build sidesteps
# all of it: a self-contained new-WoW64 10.2 (runs 32-bit without i386), extracted
# to /opt/wine. Set WINE_TARBALL_URL empty to use the winehq apt fallback instead.
# Xvfb + x11vnc are for the patched startup (tightvncserver's Xvnc lacks SHAPE).
ARG WINE_TARBALL_URL=https://github.com/Kron4ek/Wine-Builds/releases/download/10.2/wine-10.2-staging-amd64-wow64.tar.xz
# winehq apt fallback (only used when WINE_TARBALL_URL is empty):
ARG WINE_BRANCH=stable
ARG WINE_VERSION=
ARG WINE_I386=1
RUN set -eux; \
    apt-get update; \
    if [ -n "${WINE_TARBALL_URL}" ]; then \
      # Portable Wine: extract + symlink + install only the 64-bit runtime libs it needs.
      apt-get install -y --no-install-recommends \
        xz-utils ca-certificates fontconfig cabextract \
        libfreetype6 libgnutls30 libgmp10 libgl1 libglu1-mesa \
        libx11-6 libxext6 libxrandr2 libxi6 libxcursor1 libxcomposite1 libxinerama1 libxrender1 libasound2; \
      mkdir -p /opt/wine; \
      wget -O /tmp/wine.tar.xz "${WINE_TARBALL_URL}"; \
      tar -xJf /tmp/wine.tar.xz -C /opt/wine --strip-components=1; \
      rm -f /tmp/wine.tar.xz; \
      for b in /opt/wine/bin/*; do ln -sf "$b" /usr/local/bin/; done; \
      /usr/local/bin/wine --version; \
    else \
      dpkg --add-architecture i386; \
      mkdir -pm755 /etc/apt/keyrings; \
      wget -O /etc/apt/keyrings/winehq-archive.key https://dl.winehq.org/wine-builds/winehq.key; \
      wget -nc -P /etc/apt/sources.list.d/ https://dl.winehq.org/wine-builds/debian/dists/bookworm/winehq-bookworm.sources; \
      apt-get update; \
      if [ -n "${WINE_VERSION}" ] && [ "${WINE_I386}" = "0" ]; then \
        apt-get install -y --no-install-recommends \
          winehq-${WINE_BRANCH}=${WINE_VERSION} wine-${WINE_BRANCH}=${WINE_VERSION} wine-${WINE_BRANCH}-amd64=${WINE_VERSION}; \
      elif [ -n "${WINE_VERSION}" ]; then \
        apt-get install -y --install-recommends \
          winehq-${WINE_BRANCH}=${WINE_VERSION} wine-${WINE_BRANCH}=${WINE_VERSION} \
          wine-${WINE_BRANCH}-amd64=${WINE_VERSION} wine-${WINE_BRANCH}-i386=${WINE_VERSION}; \
      else \
        apt-get install -y --install-recommends winehq-${WINE_BRANCH}; \
      fi; \
      apt-get install -y cabextract winbind; \
    fi; \
    apt-get install -y xvfb x11vnc; \
    apt-get clean && rm -rf /var/lib/apt/lists/*

# Pre-cache wine-mono + wine-gecko so wineboot installs them offline (no popup,
# no per-prefix download). Best-effort (MT5 itself is native); bump the versions
# to match the Wine build if a mono/gecko prompt still appears.
ARG WINE_MONO_VERSION=9.4.0
ARG WINE_GECKO_VERSION=2.47.4
RUN mkdir -p /usr/share/wine/mono /usr/share/wine/gecko && \
    wget -O /usr/share/wine/mono/wine-mono-${WINE_MONO_VERSION}-x86.msi \
      https://dl.winehq.org/wine/wine-mono/${WINE_MONO_VERSION}/wine-mono-${WINE_MONO_VERSION}-x86.msi || true && \
    wget -O /usr/share/wine/gecko/wine-gecko-${WINE_GECKO_VERSION}-x86.msi \
      https://dl.winehq.org/wine/wine-gecko/${WINE_GECKO_VERSION}/wine-gecko-${WINE_GECKO_VERSION}-x86.msi || true && \
    wget -O /usr/share/wine/gecko/wine-gecko-${WINE_GECKO_VERSION}-x86_64.msi \
      https://dl.winehq.org/wine/wine-gecko/${WINE_GECKO_VERSION}/wine-gecko-${WINE_GECKO_VERSION}-x86_64.msi || true

# Create the xstartup script to launch XFCE immediately at VNC session start
RUN mkdir -p /root/.vnc && \
    echo '#!/bin/sh\n\
xrdb $HOME/.Xresources\n\
startxfce4 &' > /root/.vnc/xstartup && chmod +x /root/.vnc/xstartup

# Copy the startup script
COPY startup.sh /startup.sh
RUN chmod +x /startup.sh

COPY mt5-mt5debian.sh /mt5-mt5debian.sh
RUN chmod +x /mt5-mt5debian.sh

COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

COPY SocketBridgeEA.ex5 /SocketBridgeEA.ex5

# Expose the noVNC port
EXPOSE 6080

# Set the startup script as the default command
CMD ["/entrypoint.sh"]