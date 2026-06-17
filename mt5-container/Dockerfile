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

# --- Wine + display deps, baked at BUILD time (not reinstalled per container) ---
# The MetaQuotes MT5 installer is VMProtect-packed; its anti-debug trips on Wine
# stable 9.x ("A debugger has been found running in your system"). Wine 10.x /
# staging passes it. Xvfb + x11vnc are for the patched startup (tightvncserver's
# Xvnc lacks the X SHAPE extension, which crashes Wine's prefix bootstrap).
# Defaults: stable + latest (boots on the widest range of hosts — Wine 10.x's
# new WoW64 fails to bootstrap on some VPS kernels). Override at build time for a
# specific build, e.g. WINE_BRANCH=staging WINE_VERSION=10.2~bookworm-1 (10.2 is
# reported to pass the MT5 installer's anti-debug, where it boots). An EMPTY
# WINE_VERSION installs the latest of the branch — the default — so an empty
# build-arg from compose can't fall back to a nonexistent pinned version.
ARG WINE_BRANCH=stable
ARG WINE_VERSION=
RUN dpkg --add-architecture i386 && \
    mkdir -pm755 /etc/apt/keyrings && \
    wget -O /etc/apt/keyrings/winehq-archive.key https://dl.winehq.org/wine-builds/winehq.key && \
    wget -nc -P /etc/apt/sources.list.d/ https://dl.winehq.org/wine-builds/debian/dists/bookworm/winehq-bookworm.sources && \
    apt-get update && \
    if [ -n "${WINE_VERSION}" ]; then \
      apt-get install -y --install-recommends \
        winehq-${WINE_BRANCH}=${WINE_VERSION} \
        wine-${WINE_BRANCH}=${WINE_VERSION} \
        wine-${WINE_BRANCH}-amd64=${WINE_VERSION} \
        wine-${WINE_BRANCH}-i386=${WINE_VERSION} ; \
    else \
      apt-get install -y --install-recommends winehq-${WINE_BRANCH} ; \
    fi && \
    apt-get install -y xvfb x11vnc cabextract winbind && \
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