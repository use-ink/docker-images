FROM docker.io/library/debian:stable-slim

### meta ###
# Static vars, dynamic vars can be found in the build-args file. Variables were moved for caching.
ENV RESTIC_VERSION="0.16.1" \
    MINIO_VERSION="2023-04-06T16-51-10Z" \
    RUST_STABLE_VERSION=1.81.0 \
    RUST_NIGHTLY_VERSION=2024-11-28 \
    CODECOV_UPLOADER_VERSION=v0.7.3 \
    CODECOV_CLI_VERSION=v0.6.0

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

COPY ci/cargo-config /root/.cargo/config
COPY ci/cargo-config /home/nonroot/.cargo/config

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

# base | install rustup, use minimum components
RUN curl -L "https://static.rust-lang.org/rustup/dist/x86_64-unknown-linux-gnu/rustup-init" \
    -o rustup-init && \
    chmod u+x rustup-init && \
    ./rustup-init -y --no-modify-path --default-toolchain none && \
    rm -f rustup-init && \
    chown -R root:nonroot ${RUSTUP_HOME} ${CARGO_HOME} && \
    chmod -R g+w ${RUSTUP_HOME} ${CARGO_HOME}

### generic ci ####

ARG GH_TOOL_VERSION="2.54.0"

RUN wget "https://github.com/cli/cli/releases/download/v${GH_TOOL_VERSION}/gh_${GH_TOOL_VERSION}_linux_amd64.deb" && \
    dpkg -i "gh_${GH_TOOL_VERSION}_linux_amd64.deb" && rm "gh_${GH_TOOL_VERSION}_linux_amd64.deb"

ARG RUST_STABLE_VERSION="1.83.0"

# generic ci | install stable rust
RUN rustup toolchain install "${RUST_STABLE_VERSION}" --profile minimal\
      --component rustfmt clippy rust-src && \
    rustup default "${RUST_STABLE_VERSION}"

# generic ci | "alias" pinned stable toolchain as generic stable
RUN ln -s "/usr/local/rustup/toolchains/${RUST_STABLE_VERSION}-x86_64-unknown-linux-gnu" /usr/local/rustup/toolchains/stable-x86_64-unknown-linux-gnu

# generic ci | install asm toolchain for the default stable toolchain
RUN rustup target add riscv64imac-unknown-none-elf

# needed for contracts node
RUN rustup target add wasm32-unknown-unknown

# generic ci | install specific rust nightly, default is stable, use minimum components
RUN rustup toolchain install "nightly-${RUST_NIGHTLY_VERSION}" --profile minimal \
    --component rustfmt clippy rust-analyzer llvm-tools rustc-dev \
    rust-src llvm-tools-preview

# generic ci | "alias" pinned nightly toolchain as generic nightly
RUN ln -s "/usr/local/rustup/toolchains/nightly-${RUST_NIGHTLY_VERSION}-x86_64-unknown-linux-gnu" /usr/local/rustup/toolchains/nightly-x86_64-unknown-linux-gnu

RUN rustup target add riscv64imac-unknown-none-elf \
    --toolchain "nightly-${RUST_NIGHTLY_VERSION}"
#RUN rustup target add wasm32-unknown-unknown \
    #--toolchain "nightly-${RUST_NIGHTLY_VERSION}"
#RUN rustup component add rustfmt clippy rust-src rustc-dev llvm-tools-preview \
    #--toolchain "nightly-${RUST_NIGHTLY_VERSION}"
RUN rustup run nightly-${RUST_NIGHTLY_VERSION} cargo install cargo-dylint dylint-link

RUN cargo install cargo-spellcheck --locked

RUN cargo install cargo-nextest --locked

# generic ci | install zepter
RUN cargo install zepter --locked --version 0.15.0

# contracts ci
# `cargo-dylint` and `dylint-link` are dependencies needed to run `cargo-contract`.
#RUN	cargo install cargo-dylint dylint-link

# contracts ci
# Install the latest `cargo-contract`
RUN cargo install --git https://github.com/paritytech/cargo-contract \
    --locked --branch cmichi-remove-wasm-default-to-revive

# contracts ci
RUN cargo install subkey

# contracts ci
# Download the latest `substrate-contracts-node` binary
RUN cargo install --git https://github.com/use-ink/substrate-contracts-node \
    --branch upstream-master --force

# install paritytech/revive with its dependencies

RUN curl https://github.com/ethereum/solidity/releases/download/v0.8.28/solc-static-linux > /usr/bin/solc && \
    chmod u+x /usr/bin/solc && \
    wget -qO- "https://cmake.org/files/v3.20/cmake-3.20.1-linux-x86_64.tar.gz" | \
      tar --strip-components=1 -xz -C /usr/local && \
    apt-get install -y --no-install-recommends ninja-build && \
    git clone https://github.com/paritytech/revive.git --depth 1 && \
    cd revive && \
    bash build-llvm.sh && \
    export PATH=${PWD}/llvm18.0/bin:$PATH && \
    export PATH=/builds/revive/llvm18.0/bin/:$PATH && \
    make install-bin && cd ../ && rm -rf revive && \
    apt-get remove -y ninja-build && \
    rm -rf /usr/local/share/cmake-3.20

#RUN resolc --version

# ink-ci-linux
# Needed for running commands in parallel, without overlapping output.
# gnupg is only needed to verify the signature of the codecov uploader.
#RUN	apt-get install -y --no-install-recommends parallel

# We require `grcov` for coverage reporting and `rust-covfix` to improve it.
# We require `xargo` so that `miri` runs properly
RUN	cargo install grcov rust-covfix xargo

# codecov

RUN	cargo +nightly-${RUST_NIGHTLY_VERSION} install grcov rust-covfix xargo dylint-link

### codecov

RUN curl https://keybase.io/codecovsecurity/pgp_keys.asc | gpg --import

# codecov uploader
ARG CODECOV_UPLOADER_VERSION="v0.7.3"

RUN	curl --remote-name --silent https://uploader.codecov.io/${CODECOV_UPLOADER_VERSION}/linux/codecov && \
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

# finalize | cargo clean up, removes compilation artifacts cargo install creates (>250M)
#python3-minimal python3.11 python3.11-minimal
#RUN apt-get remove -y python3 cpp cpp-12 g++ g++-12 gcc gcc-12 python3-minimal python3.11 python3.11-minimal
# finalize | apt clean up
RUN rm -rf "${CARGO_HOME}/registry" "${CARGO_HOME}/git" && \
    apt-get remove -y python3 && \
    apt-get autoremove -y && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* \
    rm -rf /usr/local/doc \
    rm -rf /usr/lib/llvm-11 \
    rm -rf /tmp/*