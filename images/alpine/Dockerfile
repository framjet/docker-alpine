ARG UPTRACK_SOURCE=alpine:latest
ARG WAIT4X_VERSION=latest
FROM docker.io/framjet/wait4x:${WAIT4X_VERSION} AS wait4x
FROM ${UPTRACK_SOURCE} AS base

LABEL org.opencontainers.image.title="Alpine"
LABEL org.opencontainers.image.authors="Aurimas Niekis <aurimas@niekis.lt>"
LABEL org.opencontainers.image.source="https://github.com/framjet/docker-lighttpd"
LABEL org.opencontainers.image.licenses="MIT"

COPY --chown=root docker-entrypoint.d /docker-entrypoint.d
COPY --chown=root docker-entrypoint.sh /docker-entrypoint.sh
COPY --chown=root docker-entrypoint.functions /docker-entrypoint.functions
COPY --chown=root --from=wait4x /usr/bin/wait4x /usr/local/bin/wait4x

ENTRYPOINT ["/docker-entrypoint.sh"]

CMD ["/bin/sh"]
