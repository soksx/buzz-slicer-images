# syntax=docker/dockerfile:1
# ARM64 Slicer Codex runner. Every OCI source is pinned to a digest.
ARG SLICER_BASE_IMAGE=ghcr.io/openfaasltd/slicer-systemd-2404-arm64-avz:6.12.70-aarch64-avz-latest@sha256:855dc5c9cf60b480a20c3ca878583c272176f71abf27fe55ab1a88359b5e0434
ARG BUZZ_REF=a96af89526f7181543e7651100a944aa8e21812b
ARG BUZZ_RELAY_IMAGE=ghcr.io/block/buzz:sha-a96af89@sha256:2f7ba1afb176583b1037ea119d60597ccf1a8acace068736c6f8d6bba01369f1
ARG RUST_IMAGE=rust:1.88-bookworm@sha256:af306cfa71d987911a781c37b59d7d67d934f49684058f96cf72079c3626bfe0
ARG NODE_IMAGE=node:24-bookworm-slim@sha256:3638d9a6fe4030bd716be989438248074489337ba3275657f93595428be4fc03
ARG BUN_IMAGE=oven/bun:1.3.14@sha256:e10577f0db68676a7024391c6e5cb4b879ebd17188ab750cf10024a6d700e5c4
ARG CARGO_BUILD_JOBS=1
ARG TARGETARCH
ARG RTK_VERSION=0.45.0

FROM ${BUZZ_RELAY_IMAGE} AS buzz-relay-dist

FROM ${RUST_IMAGE} AS buzz-build
ARG BUZZ_REF
ARG CARGO_BUILD_JOBS
RUN apt-get update -qy && apt-get install -qy --no-install-recommends \
      build-essential git pkg-config libssl-dev protobuf-compiler \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /src
RUN git init . \
    && git remote add origin https://github.com/block/buzz.git \
    && git fetch --depth=1 origin "${BUZZ_REF}" \
    && git checkout --detach FETCH_HEAD \
    && cargo build --release --jobs "${CARGO_BUILD_JOBS}" -p buzz-acp -p buzz-cli -p git-credential-nostr

FROM ${NODE_IMAGE} AS node-runtime
FROM ${BUN_IMAGE} AS bun-runtime

FROM ${SLICER_BASE_IMAGE}
ARG CODEX_VERSION=0.147.0
ARG CODEX_ACP_VERSION=1.2.0
ARG TARGETARCH
ARG RTK_VERSION
COPY --from=node-runtime /usr/local/bin /usr/local/bin
COPY --from=node-runtime /usr/local/lib/node_modules /usr/local/lib/node_modules
COPY --from=bun-runtime /usr/local/bin/bun /usr/local/bin/bun
COPY --from=buzz-build /src/target/release/buzz-acp /usr/local/bin/buzz-acp
COPY --from=buzz-build /src/target/release/buzz /usr/local/bin/buzz
COPY --from=buzz-relay-dist /usr/local/bin/buzz-relay /usr/local/bin/buzz-relay
COPY --from=buzz-build /src/target/release/git-credential-nostr /usr/local/bin/git-credential-nostr

RUN apt-get update -qy && apt-get install -qy --no-install-recommends \
      ca-certificates curl git libssl3 \
    && npm install --global "@openai/codex@${CODEX_VERSION}" "@agentclientprotocol/codex-acp@${CODEX_ACP_VERSION}" \
    && case "$TARGETARCH" in \
         arm64) rtk_archive='rtk-aarch64-unknown-linux-gnu.tar.gz'; rtk_sha='80a746dd305ef944ff50ef011ae4ce3878dd5ba88dfe35d859d05498191637c3' ;; \
         *) echo "unsupported RTK architecture: $TARGETARCH" >&2; exit 1 ;; \
       esac \
    && curl -fsSL -o /tmp/rtk.tar.gz "https://github.com/rtk-ai/rtk/releases/download/v${RTK_VERSION}/${rtk_archive}" \
    && echo "$rtk_sha  /tmp/rtk.tar.gz" | sha256sum -c - \
    && tar -xzf /tmp/rtk.tar.gz -C /tmp \
    && install -m 0755 /tmp/rtk /usr/local/bin/rtk \
    && rtk --version \
    && useradd --create-home --uid 10001 --user-group --shell /bin/bash buzz \
    && install -d -o buzz -g buzz /workspace \
    && rm -rf /tmp/rtk /tmp/rtk.tar.gz /var/lib/apt/lists/*

# The image contains buzz-relay for diagnostics only. It is never enabled here:
# every agent uses the shared externally managed relay.
