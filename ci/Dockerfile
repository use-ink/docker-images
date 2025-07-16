FROM docker.io/library/debian:stable-slim

### meta ###
ENV RUST_STABLE_VERSION=1.86.0 \
    # restricted by https://github.com/trailofbits/dylint/blob/master/examples/general/rust-toolchain
    POLKADOT_SDK_BRANCH=master \
    RUST_NIGHTLY_VERSION=2025-02-20 \
    INK_NODE_VERSION=main

WORKDIR /builds

ENV SHELL=/bin/bash \
    DEBIAN_FRONTEND=noninteractive \
    RUSTUP_HOME=/usr/local/rustup \
    CARGO_HOME=/usr/local/cargo \
    PATH=/usr/local/cargo/bin:$PATH \
    RUST_BACKTRACE=1

ARG TARGETARCH
ENV ARCH=$TARGETARCH

### base ###

# base | add non-root user
RUN groupadd -g 1000 nonroot && \
    useradd -u 1000 -g 1000 -s /bin/bash -m nonroot

# base | install tools and dependencies
RUN set -eux; \
    apt-get -y update; \
    apt-get install -y --no-install-recommends \
    libssl-dev make build-essential \
    git pkg-config curl wget time rhash ca-certificates jq \
    lsof git-restore-mtime \
    bzip2 xz-utils zstd unzip gnupg protobuf-compiler \
    # needed for `paritytech/revive`
    libtinfo-dev \
    # needed for `cargo-spellcheck`
    libclang-dev && \
    # base | clean up layer
    apt-get autoremove -y && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* /usr/share/doc /usr/share/man /usr/local/doc /usr/lib/llvm-11 /tmp/* /var/tmp/*

### generic ci ####

ARG GH_TOOL_VERSION="2.54.0"

RUN wget "https://github.com/cli/cli/releases/download/v${GH_TOOL_VERSION}/gh_${GH_TOOL_VERSION}_linux_${TARGETARCH}.deb" && \
    dpkg -i "gh_${GH_TOOL_VERSION}_linux_${TARGETARCH}.deb" && \
    rm "gh_${GH_TOOL_VERSION}_linux_${TARGETARCH}.deb"

# generic ci | install rust toolchains
# llvm-tools-preview is for grcov
# base | install rustup, use minimum components
RUN case ${TARGETARCH} in \
        "amd64")  RUST_ARCH="x86_64"  ;; \
        "arm64")  RUST_ARCH="aarch64" ;; \
        *)        RUST_ARCH=${TARGETARCH} ;; \
    esac && \
    curl -L "https://static.rust-lang.org/rustup/dist/${RUST_ARCH}-unknown-linux-gnu/rustup-init" -o rustup-init && \
    chmod u+x rustup-init && \
    ./rustup-init -y --no-modify-path --default-toolchain none && \
    rm -f rustup-init && \
    chown -R root:nonroot ${RUSTUP_HOME} ${CARGO_HOME} && \
    chmod -R g+w ${RUSTUP_HOME} ${CARGO_HOME} && \
    # generic ci | install specific stable version
    rustup toolchain install "${RUST_STABLE_VERSION}" --profile minimal \
      --component rustfmt,clippy,rust-src,llvm-tools-preview && \
    rustup default "${RUST_STABLE_VERSION}" && \
    rm -rf "${CARGO_HOME}/registry" "${CARGO_HOME}/git" && \
    # generic ci | "alias" pinned stable toolchain as generic stable
    ln -s "/usr/local/rustup/toolchains/${RUST_STABLE_VERSION}-${RUST_ARCH}-unknown-linux-gnu" "/usr/local/rustup/toolchains/stable-${RUST_ARCH}-unknown-linux-gnu" && \
    # generic ci | install asm toolchain for the default stable toolchain; `wasm32v1-none` for `ink-node`
    rustup target add riscv64imac-unknown-none-elf wasm32v1-none && \
    # generic ci | install specific rust nightly, default is stable, use minimum components
    rustup toolchain install "nightly-${RUST_NIGHTLY_VERSION}" --profile minimal \
      --component rustfmt,clippy,rust-analyzer,rustc-dev,rust-src,llvm-tools-preview && \
    # generic ci | "alias" pinned nightly toolchain as rolling nightly
    ln -s "/usr/local/rustup/toolchains/nightly-${RUST_NIGHTLY_VERSION}-${RUST_ARCH}-unknown-linux-gnu" "/usr/local/rustup/toolchains/nightly-${RUST_ARCH}-unknown-linux-gnu" && \
    # generic ci | install asm toolchain for the nightly toolchain
    rustup target add riscv64imac-unknown-none-elf --toolchain "nightly" && \
    # generic ci | clean up layer
    rm -rf "${RUSTUP_HOME}/downloads" "${RUSTUP_HOME}/tmp"

# generic ci | install core packages
RUN cargo +nightly install cargo-dylint dylint-link && \
    cargo install cargo-spellcheck cargo-nextest xargo --locked && \
    cargo install zepter --locked --version 1.5.1 && \
    # Install `ink-node`
    # cargo install --force --locked --git https://github.com/use-ink/ink-node --branch main && \
    curl -L https://github.com/use-ink/ink-node/releases/download/v0.44.0/ink-node-linux.tar.gz > ink-node-linux.tar.gz && \
    tar xzfv ink-node-linux.tar.gz && \
    mv ink-node-linux/ink-node /usr/local/cargo/bin/ && \
    mv ink-node-linux/eth-rpc /usr/local/cargo/bin/ && \
    # Install `cargo-contract`
    cargo install --git https://github.com/use-ink/cargo-contract --locked --branch cmichi-release-6.0.0-alpha.2 && \
    # install `polkadot-sdk`
    git clone --depth 1 --branch ${POLKADOT_SDK_BRANCH} https://github.com/paritytech/polkadot-sdk.git && \
    cd polkadot-sdk/ && \
    cargo +nightly install --path substrate/bin/utils/subkey --locked && \
    # cargo +nightly install --path substrate/frame/revive/rpc --locked && \
    cd ../ && rm -rf polkadot-sdk/ && \
    # We require `grcov` for coverage reporting and `rust-covfix` to improve it.
    cargo +nightly install grcov rust-covfix --locked && \
    curl https://keybase.io/codecovsecurity/pgp_keys.asc | gpg --import && \
    # Download the binary release of `paritytech/revive`
    if [ "$TARGETARCH" = "amd64" ]; then \
        curl -fsSL -O https://github.com/paritytech/revive/releases/download/v0.1.0-dev.13/resolc-x86_64-unknown-linux-musl && \
        chmod +x resolc-x86_64-unknown-linux-musl && \
        mv resolc-x86_64-unknown-linux-musl /usr/local/bin/resolc; \
    else \
        echo "Skipping x86_64 specific installation of resolc for $TARGETARCH"; \
    fi && \
    # generic ci | clean up layer
    rm -rf "${CARGO_HOME}/registry" "${CARGO_HOME}/git"

# Install node.js 22.x. Used for testing revive with hardhat.
RUN curl -fsSL https://deb.nodesource.com/setup_22.x | bash - && \
    apt-get install -y nodejs && \
    node -v && npm -v && npx -v && \
    apt-get autoremove -y && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* /usr/share/doc /usr/share/man

# codecov uploader
ARG CODECOV_UPLOADER_VERSION="v0.7.3"
ARG CODECOV_CLI_VERSION="v0.6.0"

RUN curl --remote-name --silent https://uploader.codecov.io/${CODECOV_UPLOADER_VERSION}/linux/codecov && \
    curl --remote-name --silent https://uploader.codecov.io/${CODECOV_UPLOADER_VERSION}/linux/codecov.SHA256SUM && \
    curl --remote-name --silent https://uploader.codecov.io/${CODECOV_UPLOADER_VERSION}/linux/codecov.SHA256SUM.sig && \
    gpg --verify codecov.SHA256SUM.sig codecov.SHA256SUM && \
    shasum --algorithm 256 --check codecov.SHA256SUM && \
    chmod +x codecov && \
    mv codecov /usr/local/bin/codecov && \
    rm -f codecov.SHA256SUM codecov.SHA256SUM.sig && \
    # codecov-cli
    curl -Os https://cli.codecov.io/${CODECOV_CLI_VERSION}/linux/codecov && \
    curl -Os https://cli.codecov.io/${CODECOV_CLI_VERSION}/linux/codecov.SHA256SUM && \
    curl -Os https://cli.codecov.io/${CODECOV_CLI_VERSION}/linux/codecov.SHA256SUM.sig && \
    gpg --verify codecov.SHA256SUM.sig codecov.SHA256SUM && \
    shasum -a 256 -c codecov.SHA256SUM && \
    chmod +x codecov && \
    mv codecov /usr/local/bin/codecovcli && \
    rm -f codecov.SHA256SUM codecov.SHA256SUM.sig
