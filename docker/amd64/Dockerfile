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

# Install DB packages
RUN apt-get update && apt-get install -y \
    --no-install-recommends \
        libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Creates a dummy project used to grab dependencies
RUN USER=root cargo new --bin /app
WORKDIR /app

# Copies over *only* your manifests and build files
COPY ./Cargo.* ./




# Builds your dependencies and removes the
# dummy project, except the target folder
# This folder contains the compiled dependencies
RUN --mount=type=cache,target=/root/.cargo/git --mount=type=cache,target=/root/.cargo/registry \
    cargo build --release \
    && find . -not -path "./target*" -delete

# Copies the complete project
# To avoid copying unneeded files, use .dockerignore
COPY . .

# Make sure that we actually build the project
RUN touch src/main.rs

# Builds again, this time it'll just be
# your actual source files being built
RUN --mount=type=cache,target=/root/.cargo/git --mount=type=cache,target=/root/.cargo/registry \
    cargo build --release


######################## RUNTIME IMAGE  ########################
# Create a new stage with a minimal image
# because we already have a binary built
FROM debian:buster-slim



# Install needed libraries (just a test case here)
RUN apt-get update && apt-get install -y \
    --no-install-recommends \
    openssl \
    ca-certificates \
    curl \
    dumb-init \
    libpq5 \
    && rm -rf /var/lib/apt/lists/*


# Copies the files from the context (Rocket.toml file and web-vault)
# and the binary from the "build" stage to the current stage
WORKDIR /
COPY --from=build /app/target/release/dieselpgcrate .

# Configures the startup!
ENTRYPOINT ["/usr/bin/dumb-init", "--"]
CMD ["/dieselpgcrate"]
