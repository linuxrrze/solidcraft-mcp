FROM ubuntu:26.04
# solidcraft package version (e.g. 0.26.1-1); empty installs the repo's newest.
ARG SOLIDCRAFT_VERSION
ARG MCP_PROXY_VERSION
ENV UBUNTU_VERSION=26.04

ENV DEB_REPO=https://linuxrrze.github.io/deb-repos

RUN << EORUN
set -e
# Skip docs/man pages -- pure image-size trimming, has to be in place before
# the first package unpacks anything under /usr/share.
{ echo 'path-exclude=/usr/share/doc/*'; \
  echo 'path-exclude=/usr/share/man/*'; \
  echo 'path-exclude=/usr/share/groff/*'; \
  echo 'path-exclude=/usr/share/info/*'; } > /etc/dpkg/dpkg.cfg.d/01_nodoc

apt update && apt -y dist-upgrade
apt install -y ca-certificates curl python3 python3-venv tini --no-install-recommends

# Add the solidcraft repo. mirror+https: apt reads the indexes from github.io
# and fetches the .deb straight from the GitHub release assets.
mkdir -p /etc/apt/keyrings
curl -fsSL "$DEB_REPO/deb-repos.asc" -o /etc/apt/keyrings/deb-repos.asc
cat > /etc/apt/sources.list.d/solidcraft.sources << EOF
Types: deb
URIs: mirror+$DEB_REPO/solidcraft/mirrors.txt
Suites: stable
Components: main
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/deb-repos.asc
EOF
apt update

SOLIDCRAFT_PKG="solidcraft"
if [ -n "$SOLIDCRAFT_VERSION" ]; then
	SOLIDCRAFT_PKG="solidcraft=$SOLIDCRAFT_VERSION"
fi
# Headless MCP needs no display, but solidcraft links the X11/Wayland/Vulkan
# client libs and renders screenshots via Mesa's CPU Vulkan (lavapipe in
# mesa-vulkan-drivers) -- install those explicitly instead of all Recommends
# (xdg-desktop-portal, ...).
apt install -y "$SOLIDCRAFT_PKG" --no-install-recommends \
	mesa-vulkan-drivers libegl-mesa0 fonts-dejavu-core

# Make sure solidcraft version matches desired version (if one was requested)
if [ -n "$SOLIDCRAFT_VERSION" ]; then
	dpkg-query -W -f='${Version}\n' solidcraft | grep -qxF "$SOLIDCRAFT_VERSION"
fi

rm -rf /var/lib/apt/lists/*

# mcp-proxy (stdio -> streamable HTTP/SSE) in its own venv
python3 -m venv /opt/mcp-proxy
/opt/mcp-proxy/bin/pip install --no-cache-dir "mcp-proxy${MCP_PROXY_VERSION:+==$MCP_PROXY_VERSION}"
ln -s /opt/mcp-proxy/bin/mcp-proxy /usr/local/bin/mcp-proxy

# Runs under an arbitrary compose `user:` (the codingenv PUID) -- give it a
# writable HOME for solidcraft's config/scratch dirs.
mkdir -p /home/mcp /etc/mcp-proxy
chmod 1777 /home/mcp

# ubuntu:26.04 ships /usr/bin/pebble (Canonical's container init) as a plain
# unregistered binary built with Go 1.26.5, which carries unpatched HIGH CVEs
# (e.g. CVE-2026-33818). This image never runs pebble -- its entrypoint
# starts mcp-proxy directly -- so drop it instead of scanning around it.
# Re-evaluate (and re-add) if an ubuntu:26.04 point release ships pebble
# built with Go >= 1.26.6 and it becomes load-bearing.
rm -f /usr/bin/pebble
EORUN

ENV HOME=/home/mcp \
    WORK_DIR=/home/sandbox/work \
    MCP_SERVICE_HOST=solidcraft-mcp \
    MCP_SERVERS_FILE=/tmp/mcp-servers.json

COPY files/app-args.conf /etc/mcp-proxy/
COPY --chmod=755 files/mcp-servers /usr/local/bin/
COPY --chmod=755 files/entrypoint.sh /

# Probe solidcraft-cli once at build time (same mechanism as storytold-mcp):
# a build whose solidcraft-cli doesn't answer MCP fails here, not at runtime.
# The probe's WORK_DIR is a throwaway.
RUN WORK_DIR=/tmp/mcp-probe mcp-servers detect \
 && rm -rf /tmp/mcp-probe /home/mcp/* /home/mcp/.[!.]* \
 && { dpkg-query -W -f='${Package}=${Version}\n' solidcraft; \
      echo "mcp-proxy=$(/opt/mcp-proxy/bin/pip show mcp-proxy | awk '$1=="Version:"{print $2}')"; } \
    > /etc/mcp-proxy/versions.txt

WORKDIR /home/sandbox/work
EXPOSE 8080/tcp

ENTRYPOINT [ "tini", "--", "/entrypoint.sh" ]
HEALTHCHECK --start-period=60s CMD curl -fsS http://127.0.0.1:8080/status || exit 1
