# build rcon client
FROM debian:stable-slim AS rcon-builder
RUN apt-get -q update \
    && DEBIAN_FRONTEND=noninteractive apt-get -qy install build-essential

WORKDIR /src
COPY rcon/ /src
RUN make

# build factorio image
FROM debian:stable-slim
LABEL maintainer="https://github.com/factoriotools/factorio-docker"

ARG USER=factorio
ARG GROUP=factorio
ARG PUID=845
ARG PGID=845
ARG BOX64_VERSION=v0.2.4

# optionally utilize a built-in map-gen-preset (see data/base/prototypes/map-gen-presets
# if this is used, the preset will be used over any .json files supplied
# vanilla factorio provides the following presets:
# rich-resources, marathon, death-world, death-world-marathon, rail-world, ribbon-world, island
# a modded factorio example for using this:
# space-exploration
ARG PRESET

# number of retries that curl will use when pulling the headless server tarball
ARG CURL_RETRIES=8

ENV PORT=34197 \
    RCON_PORT=27015 \
    SAVES=/factorio/saves \
    PRESET="$PRESET" \
    CONFIG=/factorio/config \
    MODS=/factorio/mods \
    SCENARIOS=/factorio/scenarios \
    SCRIPTOUTPUT=/factorio/script-output \
    PUID="$PUID" \
    PGID="$PGID" \
    DLC_SPACE_AGE="true"

SHELL ["/bin/bash", "-eo", "pipefail", "-c"]

RUN apt-get -q update \
    && DEBIAN_FRONTEND=noninteractive apt-get -qy install ca-certificates curl jq pwgen xz-utils procps gettext-base file --no-install-recommends \
    && if [[ "$(uname -m)" == "aarch64" ]]; then \
        echo "installing ARM compatability layer" \
        && DEBIAN_FRONTEND=noninteractive apt-get -qy install unzip --no-install-recommends \ 
        && curl -LO https://github.com/ptitSeb/box64/releases/download/${BOX64_VERSION}/box64-GENERIC_ARM-RelWithDebInfo.zip \
        && unzip box64-GENERIC_ARM-RelWithDebInfo.zip -d /bin \
        && rm -f box64-GENERIC_ARM-RelWithDebInfo.zip \
        && chmod +x /bin/box64; \
    fi \
    && rm -rf /var/lib/apt/lists/*

RUN addgroup --system --gid "$PGID" "$GROUP" \
    && adduser --system --uid "$PUID" --gid "$PGID" --no-create-home --disabled-password --shell /bin/sh "$USER"

# version checksum of the archive to download
ARG VERSION
ARG SHA256

LABEL factorio.version=${VERSION}

ENV VERSION=${VERSION} \
    SHA256=${SHA256}

RUN set -ox pipefail \
    && if [[ "${VERSION}" == "" ]]; then \
        echo "build-arg VERSION is required" \
        && exit 1; \
    fi \
    && if [[ "${SHA256}" == "" ]]; then \
        echo "build-arg SHA256 is required" \
        && exit 1; \
    fi \
    && archive="/tmp/factorio_headless_x64_$VERSION.tar.xz" \
    && mkdir -p /opt /factorio \
    && curl -sSL "https://www.factorio.com/get-download/$VERSION/headless/linux64" -o "$archive" --retry $CURL_RETRIES \
    && echo "$SHA256  $archive" | sha256sum -c \
    || (sha256sum "$archive" && file "$archive" && exit 1) \
    && tar xf "$archive" --directory /opt \
    && chmod ugo=rwx /opt/factorio \
    && rm "$archive" \
    && ln -s "$SCENARIOS" /opt/factorio/scenarios \
    && ln -s "$SAVES" /opt/factorio/saves \
    && mkdir -p /opt/factorio/config/ \
    && chown -R "$USER":"$GROUP" /opt/factorio /factorio

COPY files/*.sh /
COPY files/config.ini /opt/factorio/config/config.ini
COPY --from=rcon-builder /src/rcon /bin/rcon

VOLUME /factorio
EXPOSE $PORT/udp $RCON_PORT/tcp
ENTRYPOINT ["/docker-entrypoint.sh"]
