FROM docker.io/library/debian:stable-slim

### meta ###
ENV RUST_STABLE_VERSION=1.85.0 \
    RUST_NIGHTLY_VERSION=2025-02-20 \
    POLKADOT_SDK_HASH=c6249dca5928c12c35c577b971277d4211c928b7

WORKDIR /builds

ENV SHELL=/bin/bash \
    DEBIAN_FRONTEND=noninteractive \
    RUSTUP_HOME=/usr/local/rustup \
    CARGO_HOME=/usr/local/cargo \
    PATH=/usr/local/cargo/bin:$PATH \
    RUST_BACKTRACE=1

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
    libclang-dev

### generic ci ####

ARG GH_TOOL_VERSION="2.54.0"

RUN wget "https://github.com/cli/cli/releases/download/v${GH_TOOL_VERSION}/gh_${GH_TOOL_VERSION}_linux_amd64.deb" && \
    dpkg -i "gh_${GH_TOOL_VERSION}_linux_amd64.deb" && rm "gh_${GH_TOOL_VERSION}_linux_amd64.deb"

# generic ci | install stable rust
# llvm-tools-preview is for grcov
# base | install rustup, use minimum components
RUN curl -L "https://static.rust-lang.org/rustup/dist/x86_64-unknown-linux-gnu/rustup-init" \
    -o rustup-init && \
    chmod u+x rustup-init && \
    ./rustup-init -y --no-modify-path --default-toolchain none && \
    rm -f rustup-init && \
    chown -R root:nonroot ${RUSTUP_HOME} ${CARGO_HOME} && \
    chmod -R g+w ${RUSTUP_HOME} ${CARGO_HOME} && \
    rustup toolchain install "${RUST_STABLE_VERSION}" --profile minimal \
      --component rustfmt,clippy,rust-src,llvm-tools-preview && \
    rustup default "${RUST_STABLE_VERSION}" && \
    rm -rf "${CARGO_HOME}/registry" "${CARGO_HOME}/git" && \
    # generic ci | "alias" pinned stable toolchain as generic stable
    ln -s "/usr/local/rustup/toolchains/${RUST_STABLE_VERSION}-x86_64-unknown-linux-gnu" /usr/local/rustup/toolchains/stable-x86_64-unknown-linux-gnu && \
    # generic ci | install asm toolchain for the default stable toolchain
    rustup target add riscv64imac-unknown-none-elf && \
    # needed for `ink-node`
    rustup target add wasm32v1-none && \
    cargo install --git https://github.com/use-ink/ink-node --branch main --force --locked && \
    # generic ci | install specific rust nightly, default is stable, use minimum components
    rustup toolchain install "nightly-${RUST_NIGHTLY_VERSION}" --profile minimal \
      --component rustfmt,clippy,rust-analyzer,llvm-tools,rustc-dev,rust-src,llvm-tools-preview && \
    rustup toolchain install "nightly-2025-03-13" --profile minimal \
      --component rustfmt,clippy,rust-analyzer,llvm-tools,rustc-dev,rust-src,llvm-tools-preview && \
    # generic ci | "alias" pinned nightly toolchain as generic nightly
    rustup target add riscv64imac-unknown-none-elf \
      --toolchain "nightly-2025-03-13" && \
    rustup default nightly-2025-03-13 && \
    rustup target add riscv64imac-unknown-none-elf \
      --toolchain "nightly-${RUST_NIGHTLY_VERSION}" && \
    rustup run nightly-${RUST_NIGHTLY_VERSION} cargo install cargo-dylint dylint-link && \
    cargo install cargo-spellcheck --locked && \
    cargo install cargo-nextest --locked && \
    cargo install zepter --locked --version 1.5.1 && \
    cargo install --git https://github.com/paritytech/cargo-contract \
      --locked --branch master && \
    git clone https://github.com/paritytech/polkadot-sdk.git --depth 50 && \
    cd polkadot-sdk/ && \
    git reset --hard ${POLKADOT_SDK_HASH} && \
    cargo +nightly install --path substrate/bin/utils/subkey --locked && \
    cargo +nightly install --path substrate/frame/revive/rpc --locked && \
    cd ../ && rm -rf polkadot-sdk/ && \
    # We require `grcov` for coverage reporting and `rust-covfix` to improve it.
    # We require `xargo` so that `miri` runs properly
    cargo install grcov rust-covfix xargo && \
    # codecov
    cargo +nightly install grcov rust-covfix xargo dylint-link && \
    curl https://keybase.io/codecovsecurity/pgp_keys.asc | gpg --import && \
    # Download the binary release of `paritytech/revive`
    curl -L https://github.com/paritytech/revive/releases/download/v0.1.0-dev.12/resolc-x86_64-unknown-linux-musl.tar.gz > resolc.tar.gz && \
    tar -xvzf resolc.tar.gz && \
    rm resolc.tar.gz && \
    chmod +x resolc-x86_64-unknown-linux-musl && \
    mv resolc-x86_64-unknown-linux-musl /usr/local/bin/resolc

# Install node.js 22.x. Used for testing revive with hardhat.
RUN curl -fsSL https://deb.nodesource.com/setup_22.x | bash - && \
    apt-get install -y nodejs && \
    node -v && npm -v && npx -v && \
    apt-get autoremove -y && \
    apt-get clean

# codecov uploader
ARG CODECOV_UPLOADER_VERSION="v0.7.3"

RUN curl --remote-name --silent https://uploader.codecov.io/${CODECOV_UPLOADER_VERSION}/linux/codecov && \
    curl --remote-name --silent https://uploader.codecov.io/${CODECOV_UPLOADER_VERSION}/linux/codecov.SHA256SUM && \
    curl --remote-name --silent https://uploader.codecov.io/${CODECOV_UPLOADER_VERSION}/linux/codecov.SHA256SUM.sig && \
    gpg --verify codecov.SHA256SUM.sig codecov.SHA256SUM && \
    shasum --algorithm 256 --check codecov.SHA256SUM && \
    chmod +x codecov && \
    mv codecov /usr/local/bin/codecov && \
    rm -f codecov.SHA256SUM codecov.SHA256SUM.sig

# codecov-cli
ARG CODECOV_CLI_VERSION="v0.6.0"

RUN curl -Os https://cli.codecov.io/${CODECOV_CLI_VERSION}/linux/codecov && \
    curl -Os https://cli.codecov.io/${CODECOV_CLI_VERSION}/linux/codecov.SHA256SUM && \
    curl -Os https://cli.codecov.io/${CODECOV_CLI_VERSION}/linux/codecov.SHA256SUM.sig && \
    gpg --verify codecov.SHA256SUM.sig codecov.SHA256SUM && \
    shasum -a 256 -c codecov.SHA256SUM && \
    chmod +x codecov && \
    mv codecov /usr/local/bin/codecovcli && \
    rm -f codecov.SHA256SUM codecov.SHA256SUM.sig

### finalize ###

# finalize | apt clean up
RUN rm -rf "${CARGO_HOME}/registry" "${CARGO_HOME}/git" && \
    apt-get remove -y python3 && \
    apt-get autoremove -y && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* \
    rm -rf /usr/local/doc \
    rm -rf /usr/lib/llvm-11 \
    rm -rf /tmp/*
