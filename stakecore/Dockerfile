# syntax=docker/dockerfile:1
# check=skip=UndefinedVar,UserExist # setpriv in the entrypoint replaces the USER directive

# A headless zebrad built from this monorepo, for running a Crosslink node as a
# service. Not upstream's: zebra-crosslink/docker/Dockerfile is inherited from
# Zebra and takes zebra-crosslink/ as its build context, which cannot see the
# sibling crates the workspace depends on (../wallet, ../librustzcash, ../zebra-gui,
# ../clay-rs, ../tenderlink, ../patches). This one takes the REPOSITORY ROOT as its
# context and reuses upstream's runtime layout and entrypoint unchanged.
#
#   docker build -f stakecore/Dockerfile -t zebra-crosslink:dev .
#
# The context is cut down by stakecore/Dockerfile.dockerignore, which BuildKit picks
# up because it sits next to this file under the same name.

# Keep in sync with zebra-crosslink/rust-toolchain.toml: a mismatch is not an error,
# rustup just downloads the pinned toolchain on top of the image's.
ARG RUST_VERSION=1.94.0

ARG UID=10001
ARG GID=${UID}
ARG USER="zebra"
ARG HOME="/home/${USER}"

# ── build ────────────────────────────────────────────────────────────────────
FROM rust:${RUST_VERSION}-trixie AS build
SHELL ["/bin/bash", "-xo", "pipefail", "-c"]

# libclang for bindgen (rocksdb, zcash_script), protoc for the gRPC and BFT protos:
# the same two packages upstream's Dockerfile adds. Nothing in zebrad's Linux
# dependency tree asks for cmake (`cargo tree -p zebrad -i cmake` prints nothing).
RUN apt-get -qq update && \
    apt-get -qq install -y --no-install-recommends \
    libclang-dev \
    protobuf-compiler \
    && rm -rf /var/lib/apt/lists/*

# The workspace's release profile is `debug = "full"`, for upstream's own server
# debugging: a ~1 GB binary and a target directory that does not fit a hosted CI
# runner. Line tables keep file:line in the backtrace a `panic = "abort"` build
# prints on its way down, which is the part an operator reads.
ARG CARGO_PROFILE_RELEASE_DEBUG=line-tables-only
ENV CARGO_PROFILE_RELEASE_DEBUG=${CARGO_PROFILE_RELEASE_DEBUG}
ENV CARGO_INCREMENTAL=0

# Baked into `zebrad --version` (option_env!("SHORT_SHA")). The context carries no
# .git, so vergen's own git lookup finds nothing and says so in the build log.
ARG SHORT_SHA

WORKDIR /src
COPY . .

# Default features, as upstream's release builds use: the GUI is compiled into every
# zebrad and --headless is what keeps it closed. The target directory is a cache
# mount, so only the binary lands in a layer.
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=/usr/local/cargo/git \
    --mount=type=cache,target=/src/zebra-crosslink/target \
    cd zebra-crosslink && \
    cargo build --locked --release --package zebrad --bin zebrad && \
    cp target/release/zebrad /usr/local/bin/zebrad

# ── runtime ──────────────────────────────────────────────────────────────────
FROM debian:trixie-slim AS runtime

ARG UID
ENV UID=${UID}
ARG GID
ENV GID=${GID}
ARG USER
ENV USER=${USER}
ARG HOME
ENV HOME=${HOME}

# curl for poking the health and RPC ports from inside the container.
RUN apt-get -q update && \
    apt-get -q install -y --no-install-recommends \
    adduser \
    ca-certificates \
    curl \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/log/* /var/cache/ldconfig/aux-cache

RUN addgroup --quiet --gid ${GID} ${USER} && \
    adduser --quiet --gid ${GID} --uid ${UID} --home ${HOME} ${USER} --disabled-password --gecos ""

WORKDIR ${HOME}
RUN chown -R ${UID}:${GID} ${HOME}

COPY --from=build /usr/local/bin/zebrad /usr/local/bin/zebrad

# Upstream's entrypoint, as is: it starts as root, creates and chowns
# ZEBRA_STATE__CACHE_DIR (default ${HOME}/.cache/zebra) for the user above, then
# drops to it with setpriv. A mounted data directory therefore only has to exist.
COPY --chmod=0755 zebra-crosslink/docker/entrypoint.sh /usr/local/bin/entrypoint.sh

# A missing shared library fails the build here instead of the first deploy.
RUN zebrad --version

ENTRYPOINT [ "entrypoint.sh" ]
# --headless: without it `start` tries to open the visualizer window first.
CMD [ "zebrad", "--headless", "start" ]
