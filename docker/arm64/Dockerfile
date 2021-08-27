# syntax=docker/dockerfile:1

# This file was generated using a Jinja2 template.
# Please make your changes in `Dockerfile.j2` and then `make` the individual Dockerfiles.

# Using multistage build:
# 	https://docs.docker.com/develop/develop-images/multistage-build/
# 	https://whitfin.io/speeding-up-rust-docker-builds/

########################## BUILD IMAGE  ##########################
FROM rust:1.54 as build

ENV CARGO_HOME "/root/.cargo"
ENV USER "root"

# Build time options to avoid dpkg warnings and help with reproducible builds.
ENV DEBIAN_FRONTEND=noninteractive LANG=C.UTF-8 TZ=UTC TERM=xterm-256color

# Don't download rust docs
RUN --mount=type=cache,target=/root/.cargo/git --mount=type=cache,target=/root/.cargo/registry \
    rustup set profile minimal

# Install required build libs for arm64 architecture.
# To compile both mysql and postgresql we need some extra packages for both host arch and target arch
RUN sed 's/^deb/deb-src/' /etc/apt/sources.list > \
        /etc/apt/sources.list.d/deb-src.list \
    && dpkg --add-architecture arm64 \
    && apt-get update \
    && apt-get install -y \
        libssl-dev:arm64 \
        libc6-dev:arm64 \
        libpq5:arm64 \
        libpq-dev \
        gcc-aarch64-linux-gnu \
        --no-install-recommends \
    && echo '[target.aarch64-unknown-linux-gnu]' >> /root/.cargo/config \
    && echo 'linker = "aarch64-linux-gnu-gcc"' >> /root/.cargo/config \
    && echo 'rustflags = ["-L/usr/lib/aarch64-linux-gnu"]' >> /root/.cargo/config

# Creates a dummy project used to grab dependencies
RUN USER=root cargo new --bin /app
WORKDIR /app

# Copies over *only* your manifests and build files
COPY ./Cargo.* ./


# Configure cross-compile on default rust build image
ENV CC_aarch64_unknown_linux_gnu="/usr/bin/aarch64-linux-gnu-gcc"
ENV CROSS_COMPILE="1"
ENV OPENSSL_INCLUDE_DIR="/usr/include/aarch64-linux-gnu"
ENV OPENSSL_LIB_DIR="/usr/lib/aarch64-linux-gnu"

# For Diesel-RS migrations_macros to compile with PostgreSQL we need to do some magic.
# The libpq5:arm64 package seems to not provide a symlink to libpq.so.5 with the name libpq.so.
# This is only provided by the libpq-dev package which can't be installed for both arch at the same time.
# Without this specific file the ld command will fail and compilation fails with it.
RUN ln -sfnr /usr/lib/aarch64-linux-gnu/libpq.so.5 /usr/lib/aarch64-linux-gnu/libpq.so

RUN --mount=type=cache,target=/root/.cargo/git --mount=type=cache,target=/root/.cargo/registry \
    rustup target add aarch64-unknown-linux-gnu

# Builds your dependencies and removes the
# dummy project, except the target folder
# This folder contains the compiled dependencies
RUN --mount=type=cache,target=/root/.cargo/git --mount=type=cache,target=/root/.cargo/registry \
    cargo build --release --target=aarch64-unknown-linux-gnu \
    && find . -not -path "./target*" -delete

# Copies the complete project
# To avoid copying unneeded files, use .dockerignore
COPY . .

# Make sure that we actually build the project
RUN touch src/main.rs

# Builds again, this time it'll just be
# your actual source files being built
RUN --mount=type=cache,target=/root/.cargo/git --mount=type=cache,target=/root/.cargo/registry \
    cargo build --release --target=aarch64-unknown-linux-gnu


######################## RUNTIME IMAGE  ########################
# Create a new stage with a minimal image
# because we already have a binary built
FROM balenalib/aarch64-debian:buster


# hadolint ignore=DL3059
RUN [ "cross-build-start" ]

# Install needed libraries (just a test case here)
RUN apt-get update && apt-get install -y \
    --no-install-recommends \
    openssl \
    ca-certificates \
    curl \
    dumb-init \
    libpq5 \
    && rm -rf /var/lib/apt/lists/*

# hadolint ignore=DL3059
RUN [ "cross-build-end" ]

# Copies the files from the context (Rocket.toml file and web-vault)
# and the binary from the "build" stage to the current stage
WORKDIR /
COPY --from=build /app/target/aarch64-unknown-linux-gnu/release/dieselpgcrate .

# Configures the startup!
ENTRYPOINT ["/usr/bin/dumb-init", "--"]
CMD ["/dieselpgcrate"]
