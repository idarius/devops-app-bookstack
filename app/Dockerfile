# Pin BookStack v25.12.2
ARG BOOKSTACK_VERSION=25.12.2
FROM lscr.io/linuxserver/bookstack:version-v${BOOKSTACK_VERSION}

ARG BUILD_SHA=dev
LABEL org.opencontainers.image.revision=$BUILD_SHA

RUN echo "${BUILD_SHA}" > /build_sha.txt
