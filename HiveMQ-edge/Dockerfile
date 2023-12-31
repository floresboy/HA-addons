# We use multi-stage here to unzip in an initial layer so we don't have to COPY and then RUN unzip (two layers). ADD can lead to larger layers as well.
FROM busybox as unpack-stage

ARG HIVEMQ_EDGE_VERSION=2023.9

RUN mkdir /opt \
    && wget https://github.com/hivemq/hivemq-edge/releases/download/2023.9/hivemq-edge-2023.9.zip -P /opt \
    && unzip /opt/hivemq-edge-${HIVEMQ_EDGE_VERSION}.zip  -d /opt/ \
    && chgrp -R 0 /opt \
    && chmod -R 770 /opt

# ################## Final stage build ##################

FROM ghcr.io/hassio-addons/ubuntu-base:8.2.0

# Set shell
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

ARG HIVEMQ_EDGE_VERSION=2023.9
ENV HIVEMQ_GID=10000
ENV HIVEMQ_UID=10000

# Additional JVM options, may be overwritten by user
ENV JAVA_OPTS "-XX:+UnlockExperimentalVMOptions -XX:+UseNUMA"

# Default allow all extension, set this to false to disable it
ENV HIVEMQ_ALLOW_ALL_CLIENTS "true"

# gosu for root step-down to user-privileged process
ENV GOSU_VERSION 1.11

# Set locale
ENV LANG=en_US.UTF-8

# Setup base system
RUN \
    apt-get update \
    && apt-get install -y --no-install-recommends \
    && apt-get update && apt-get install -y --no-install-recommends curl gnupg-agent gnupg dirmngr unzip

# Setup Java env
RUN \
    apt-get install -y --no-install-recommends openjdk-17-jdk-headless=17* 



# gosu setup
RUN set -x \
        && apt-get install -y --no-install-recommends \
        && curl -fSL "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(dpkg --print-architecture)" -o /usr/local/bin/gosu \
        && curl -fSL "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(dpkg --print-architecture).asc" -o /usr/local/bin/gosu.asc \
        && export GNUPGHOME="$(mktemp -d)" \
        && gpg --batch --keyserver hkps://keyserver.ubuntu.com --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4 \
        && gpg --batch --verify /usr/local/bin/gosu.asc /usr/local/bin/gosu \
        && rm -rf "$GNUPGHOME" /usr/local/bin/gosu.asc \
        && { command -v gpgconf && gpgconf --kill all || :; } \
        && chmod +x /usr/local/bin/gosu \
        && gosu nobody true \
        && apt-get purge -y gpg dirmngr && rm -rf /var/lib/apt/lists/* \
        && mkdir -p /docker-entrypoint.d


# copy expanded Hive zipfile from prebuild stage 'unpack-stage'
COPY --from=unpack-stage /opt/hivemq-edge-${HIVEMQ_EDGE_VERSION} /opt/hivemq-edge-${HIVEMQ_EDGE_VERSION}

COPY config.xml /opt/hivemq-edge-${HIVEMQ_EDGE_VERSION}/conf/config.xml
COPY docker-entrypoint.sh /opt/docker-entrypoint.sh

RUN ln -s /opt/hivemq-edge-${HIVEMQ_EDGE_VERSION} /opt/hivemq \
    && groupadd --gid ${HIVEMQ_GID} hivemq \
    && useradd -g hivemq -d /opt/hivemq -s /bin/bash --uid ${HIVEMQ_UID} hivemq \
    && chgrp 0 /opt/hivemq-edge-${HIVEMQ_EDGE_VERSION}/conf/config.xml \
    && chmod 770 /opt/hivemq-edge-${HIVEMQ_EDGE_VERSION}/conf/config.xml \
    && chgrp 0 /opt/hivemq \
    && chmod 770 /opt/hivemq \
    && chmod +x /opt/hivemq/bin/run.sh /opt/docker-entrypoint.sh

# Persist log data
VOLUME /opt/hivemq/log

#mqtt-clients
EXPOSE 1883

#mqtt-sn-clients
EXPOSE 2442

#api and webinterface
EXPOSE 8080

WORKDIR /opt/hivemq

ENTRYPOINT ["/opt/docker-entrypoint.sh"]
CMD ["/opt/hivemq/bin/run.sh"]

#healthcheck
HEALTHCHECK CMD curl --fail http://127.0.0.1:8080 || exit 1

# Build arguments
ARG BUILD_ARCH
ARG BUILD_DATE
ARG BUILD_DESCRIPTION
ARG BUILD_NAME
ARG BUILD_REF
ARG BUILD_REPOSITORY
ARG BUILD_VERSION

# Labels
LABEL \
    io.hass.name="${BUILD_NAME}" \
    io.hass.description="${BUILD_DESCRIPTION}" \
    io.hass.arch="${BUILD_ARCH}" \
    io.hass.type="addon" \
    io.hass.version=${BUILD_VERSION} \
    maintainer="Kamiel Straatman " \
    io.hass.name="${BUILD_NAME}" \
    io.hass.description="${BUILD_DESCRIPTION}" \
    org.opencontainers.image.vendor="HA KS Add-ons" \
    org.opencontainers.image.authors="kamiel@gmail.com" \
    org.opencontainers.image.licenses="APACHE" \
    org.opencontainers.image.url="" \
    org.opencontainers.image.source="https://github.com/${BUILD_REPOSITORY}" \
    org.opencontainers.image.documentation="https://github.com/${BUILD_REPOSITORY}/blob/main/README.md" \
    org.opencontainers.image.created=${BUILD_DATE} \
    org.opencontainers.image.revision=${BUILD_REF} \
    org.opencontainers.image.version=${BUILD_VERSION}