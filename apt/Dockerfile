# Multi-stage, shared image for Ubuntu and Debian.
#
# A single Dockerfile covers all supported Ubuntu and Debian versions.
# DISTRO and VERSION are the only per-image differences, plus a handful of
# version-specific Qt packages selected at build time.
#
# Stages / build targets:
#   base     OpenModelica build dependencies only (apt build-dep openmodelica)
#   venv     Python virtualenv at /opt/venv, built in isolation and copied later
#   full     the published BASE image: base + User's Guide toolchain + Qt + python
#   cmake-4  ADD-ON: full + CMake 4 from the official Kitware release
#   debug    ADD-ON: full + GDB, Valgrind, and other debugging tools
#   rust     ADD-ON: full + Rust toolchain + Emscripten + Qt for Wasm (Ubuntu only)
#
# Build the base image (the `full` target):
#   docker build --target full --build-arg DISTRO=ubuntu --build-arg VERSION=26.04 \
#     --tag build-deps:ubuntu-26.04 apt
#   docker build --target full --build-arg DISTRO=debian --build-arg VERSION=13 \
#     --tag build-deps:debian-13 apt
#
# Build the CMake 4 add-on image (the `cmake-4` target):
#   docker build --target cmake-4 --build-arg DISTRO=ubuntu --build-arg VERSION=26.04 \
#     --tag build-deps:ubuntu-26.04-cmake-4 apt
#
# CI passes DISTRO, VERSION and the --target per image from .ci/matrix.yml.

ARG DISTRO=ubuntu
ARG VERSION=26.04

# ── base: OpenModelica build dependencies only ────────────────────────────────
FROM ${DISTRO}:${VERSION} AS base

# Image / OCI metadata (inherited by the stages built FROM this one)
LABEL maintainer="AnHeuermann" \
      description="OpenModelica build-deps Docker Image" \
      organization="OpenModelica"

LABEL org.opencontainers.image.vendor="OpenModelica" \
      org.opencontainers.image.authors="AnHeuermann" \
      org.opencontainers.image.description="OpenModelica build-deps base image" \
      org.opencontainers.image.source="https://github.com/OpenModelica/build-deps" \
      org.opencontainers.image.license="MIT"

ENV SHELL=/bin/bash \
    LANGUAGE=en_US:en \
    LANG=C.UTF-8 \
    LC_ALL=C.UTF-8

RUN export DEBIAN_FRONTEND=noninteractive && \
  apt-get update \
  && apt-get upgrade -qy \
  && apt-get dist-upgrade -qy \
  && apt-get install -qy --no-install-recommends \
    ca-certificates \
    curl \
    gnupg \
    lsb-release \
  && curl -fsSL https://build.openmodelica.org/apt/openmodelica.asc | gpg --dearmor -o /usr/share/keyrings/openmodelica-keyring.gpg \
  && echo \
    "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/openmodelica-keyring.gpg] https://build.openmodelica.org/apt \
    $(cat /etc/os-release | grep "\(UBUNTU\\|DEBIAN\\|VERSION\)_CODENAME" | sort | cut -d= -f 2 | head -1) \
    nightly" | tee /etc/apt/sources.list.d/openmodelica.list > /dev/null \
  && echo \
    "deb-src [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/openmodelica-keyring.gpg] https://build.openmodelica.org/apt \
    $(cat /etc/os-release | grep "\(UBUNTU\\|DEBIAN\\|VERSION\)_CODENAME" | sort | cut -d= -f 2 | head -1) \
    nightly" | tee -a /etc/apt/sources.list.d/openmodelica.list > /dev/null \
  && apt-get update \
  && apt-get build-dep -qy openmodelica \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/*

# CI mounts fresh named volumes under /cache; Docker seeds an empty volume with
# the mount point's image permissions, so create them world-writable for the
# non-root Jenkins uid (emscripten/sccache/runtest/omlibrary caches).
RUN mkdir -p /cache/emscripten /cache/sccache /cache/runtest /cache/omlibrary \
  && chmod -R 0777 /cache

# ── venv: Python tooling, isolated so it caches independently ─────────────────
# Built once and copied into `full`. Uses the same python3 as `full` (same
# base image), so the relocated /opt/venv works there unchanged.
FROM base AS venv
RUN apt-get update \
  && apt-get install -qy --no-install-recommends python3-pip python3-venv \
  && rm -rf /var/lib/apt/lists/*
RUN python3 -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
# Use permalink for doc/UsersGuide/source/requirements.txt to keep builds
# deterministic.
RUN pip install --no-cache-dir \
    fmpy \
    junit_xml \
    lxml \
    ompython==3.6.0 \
    PyGithub \
    simplejson \
    svgwrite \
  && pip install --no-cache-dir -r \
    https://raw.githubusercontent.com/OpenModelica/OpenModelica/9c0dc9a8ab50ba652109584cb3fecaef86640b66/doc/UsersGuide/source/requirements.txt

# ── rustdeps: Dependencies for the Rust build (Ubuntu only) ───────────────────
# Built once and copied into `rust`. Not used for Debian images.
FROM base AS rustdeps
RUN apt-get update \
  && apt-get install --no-install-recommends -qy python3-pip python3-venv curl \
  && rm -rf /var/lib/apt/lists/*

# Specific versions needed for caching Rust crates
ARG WASM_BINDGEN_VERSION="0.2.125"
ARG RUST_NIGHTLY="nightly-2026-05-31"
# Upstream binaryen: the crates.io wasm-opt is stuck at binaryen 116 and Ubuntu's
# apt build is 120; wasm-bindgen is newer and emits newer opcodes.
ARG BINARYEN_VERSION="version_130"
ARG BINARYEN_SHA256="0a18362361ad05465118cd8eeb72edaeec89de6894bc283576ef4e07aa3babcc"

ENV RUSTUP_HOME=/opt/rust/rustup \
  CARGO_HOME=/opt/rust/cargo \
  XWIN_CACHE_DIR=/opt/cargo-xwin

# Install Rust
RUN curl https://sh.rustup.rs | bash -s -- -y \
      --profile minimal \
      --default-toolchain "${RUST_NIGHTLY}" \
      --target x86_64-pc-windows-msvc \
      --target aarch64-pc-windows-msvc \
      --target wasm32-unknown-unknown && \
    . "$CARGO_HOME/env" && \
    which cargo && cargo --version && \
    rustup component add rustc-codegen-cranelift-preview clippy rustfmt && \
    cargo install wasm-bindgen-cli --version "${WASM_BINDGEN_VERSION}" && \
    cargo install sccache --locked && \
    cargo install cargo-nextest --locked && \
    curl -sSfL "https://github.com/WebAssembly/binaryen/releases/download/${BINARYEN_VERSION}/binaryen-${BINARYEN_VERSION}-x86_64-linux.tar.gz" -o /tmp/binaryen.tar.gz && \
    echo "${BINARYEN_SHA256}  /tmp/binaryen.tar.gz" | sha256sum -c - && \
    tar -xzf /tmp/binaryen.tar.gz -C /tmp && \
    install -m755 "/tmp/binaryen-${BINARYEN_VERSION}/bin/wasm-opt" "$CARGO_HOME/bin/wasm-opt" && \
    rm -rf /tmp/binaryen.tar.gz "/tmp/binaryen-${BINARYEN_VERSION}" && \
    cargo install --locked cargo-xwin && \
    cargo xwin cache xwin \
      --xwin-arch x86_64,aarch64 && \
    rm -rf "$CARGO_HOME/registry" "$CARGO_HOME/git" \
           "$CARGO_HOME/.package-cache" \
           "$HOME/.rustup/downloads" "$HOME/.rustup/tmp" \
           "${CARGO_TARGET_DIR:-/nonexistent}" && \
    mkdir -p "$CARGO_HOME/registry" && chmod ugo+rwx -R /opt/rust

RUN export DEBIAN_FRONTEND=noninteractive && \
    apt-get update && \
    apt-get install -y --no-install-recommends \
        git python3 python3-pip python3-venv ca-certificates xz-utils flang llvm && \
    # --- Emscripten 4.0.7 (required by Qt 6.10.x; Ubuntu has 3.1.69) ---
    git clone --depth 1 https://github.com/emscripten-core/emsdk.git /opt/emsdk; \
    cd /opt/emsdk && \
    ./emsdk install 4.0.7 && \
    ./emsdk activate 4.0.7 && \
    # --- Qt 6.10.2: WebAssembly (single-threaded) + matching 26.04 host ---
    python3 -m pip install --no-cache-dir --break-system-packages aqtinstall && \
    aqt install-qt linux desktop 6.10.2 linux_gcc_64 -m qtwebchannel qtpositioning qtpdf qthttpserver qtquick3d qtshadertools -O /opt/Qt && \
    aqt install-qt all_os wasm 6.10.2 wasm_singlethread -m qt5compat qtquick3d qtshadertools -O /opt/Qt && \
    aqt install-qt windows desktop 6.10.2 win64_msvc2022_64 -m qtwebchannel qtpositioning qtpdf qthttpserver qtquick3d qtshadertools -O /opt/Qt && \
    rm -rf /var/lib/apt/lists/* && \
    chmod +x /opt/Qt/6.10.2/wasm_singlethread/bin/qt-cmake \
         /opt/Qt/6.10.2/wasm_singlethread/bin/qt-cmake-create \
         /opt/Qt/6.10.2/wasm_singlethread/bin/qt-configure-module

# ── full: the published base image ────────────────────────────────────────────
#   - tools to build the User's Guide (common to all versions)
#   - Qt5/Qt6 packages selected by distro and version:
#       ubuntu:22.04  older Qt6 (Jammy), lacks some packages added in 24.04+
#       debian:12     Bookworm Qt6, lacks qt6-httpserver-dev (added in Trixie)
#       *             Ubuntu 24.04+ and Debian 13+
FROM base AS full

# Package lists as build-args, so they are easy to read and to override:
#   - COMMON_PKGS      : tools common to all distro versions (User's Guide, …)
#   - QT_PKGS_22_04    : Qt set for Ubuntu Jammy (older Qt6)
#   - QT_PKGS_DEBIAN_12: Qt set for Debian Bookworm (missing qt6-httpserver-dev)
#   - QT_PKGS          : Qt set for Ubuntu 24.04+ and Debian 13+ (the default)
#   - INTEL_OCL_PKGS   : Intel OpenCL ICD; empty for Debian 13 (not in Trixie).
#                        ocl-icd-opencl-dev + pocl-opencl-icd in COMMON_PKGS
#                        still provide the ICD loader and a CPU runtime.
ARG COMMON_PKGS="\
    aspell \
    bibtex2html \
    bison \
    ccache \
    clang-tools \
    devscripts \
    docker.io \
    doxygen \
    equivs \
    flex \
    git \
    gnuplot-nox \
    inkscape \
    jq \
    latexmk \
    libcurl4-gnutls-dev \
    libmldbm-perl \
    locales \
    ocl-icd-opencl-dev \
    opencl-headers \
    pandoc \
    pocl-opencl-icd \
    poppler-utils \
    python3-pip \
    python3-venv \
    subversion \
    texlive-base \
    texlive-bibtex-extra \
    texlive-lang-greek \
    texlive-latex-extra \
    unzip \
    wget \
    xsltproc \
    xvfb \
    zip"
ARG QT_PKGS_22_04="\
    libqt6core5compat6-dev \
    libqt6opengl6-dev \
    libqt6openglwidgets6 \
    libqt6svg6-dev \
    qt6-base-dev \
    qt6-l10n-tools \
    qt6-tools-dev \
    qt6-tools-dev-tools \
    qt6-webengine-dev \
    qt6-webengine-dev-tools \
    qt6-quick3d-dev \
    qttools5-dev \
    qtwebengine5-dev"
ARG QT_PKGS_DEBIAN_12="\
    libqt6core5compat6-dev \
    libqt6opengl6-dev \
    libqt6openglwidgets6 \
    libqt6svg6-dev \
    qt6-base-dev \
    qt6-scxml-dev \
    qt6-tools-dev \
    qt6-tools-dev-tools \
    qt6-webengine-dev \
    qt6-websockets-dev \
    qt6-quick3d-dev \
    qtwebengine5-dev"
ARG QT_PKGS="\
    libqt6core5compat6-dev \
    libqt6opengl6-dev \
    libqt6openglwidgets6 \
    libqt6svg6-dev \
    qt6-base-dev \
    qt6-httpserver-dev \
    qt6-scxml-dev \
    qt6-tools-dev \
    qt6-tools-dev-tools \
    qt6-webengine-dev \
    qt6-websockets-dev \
    qt6-quick3d-dev \
    qtwebengine5-dev"
ARG INTEL_OCL_PKGS="intel-opencl-icd"
RUN export DEBIAN_FRONTEND=noninteractive && \
  apt-get update \
  && . /etc/os-release \
  && case "${ID}:${VERSION_ID}" in \
       ubuntu:22.04) qt="${QT_PKGS_22_04}" ;; \
       debian:12)    qt="${QT_PKGS_DEBIAN_12}" ;; \
       *)            qt="${QT_PKGS}" ;; \
     esac \
  && apt-get install -qy ${COMMON_PKGS} ${qt} ${INTEL_OCL_PKGS} \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/*

# Pull in the venv built in its own stage instead of rebuilding it here.
COPY --from=venv /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# ── cmake-4 add-on: full + CMake 4 from the official Kitware release ──────────
# Overrides any apt-provided cmake. Built with `--target cmake-4`.
FROM full AS cmake-4
LABEL org.opencontainers.image.description="OpenModelica build-deps with latest CMake 4"
ARG CMAKE_VERSION=4.3.4
RUN curl -fsSL "https://github.com/Kitware/CMake/releases/download/v${CMAKE_VERSION}/cmake-${CMAKE_VERSION}-linux-$(uname -m).sh" \
      -o /tmp/cmake-install.sh \
  && chmod +x /tmp/cmake-install.sh \
  && /tmp/cmake-install.sh --skip-license --prefix=/usr/local \
  && rm /tmp/cmake-install.sh

# ── debug add-on: full + GDB, Valgrind, and other debugging tools ─────────────
# Overrides nothing in full; adds debuggers and profilers on top.
# Built with `--target debug`.
FROM full AS debug
LABEL org.opencontainers.image.description="OpenModelica build-deps with debug tools"
RUN export DEBIAN_FRONTEND=noninteractive && \
  apt-get update \
  && . /etc/os-release \
  && case "${ID}" in \
       ubuntu) linux_perf="linux-tools-generic" ;; \
       *)      linux_perf="linux-perf" ;; \
     esac \
  && apt-get install -qy \
    gdb \
    valgrind \
    ${linux_perf} \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/*

# ── rust add-on: full + Rust toolchain + Emscripten + Qt for Wasm ─────────────
# Ubuntu only (uses flang and Ubuntu-specific Qt Wasm build).
# Built with `--target rust`.
FROM full AS rust
LABEL org.opencontainers.image.description="OpenModelica build-deps (Ubuntu) for the Rust targets"

ENV EMSDK=/opt/emsdk \
  QT_HOST_PATH=/opt/Qt/6.10.2/gcc_64 \
  RUSTUP_HOME=/opt/rust/rustup \
  CARGO_HOME=/opt/rust/cargo \
  XWIN_CACHE_DIR=/opt/cargo-xwin \
  PATH=/opt/emsdk:/opt/emsdk/upstream/emscripten:/opt/Qt/6.10.2/wasm_singlethread/bin:/opt/rust/cargo/bin:$PATH

COPY --from=rustdeps /opt/emsdk /opt/emsdk
COPY --from=rustdeps /opt/Qt /opt/Qt
COPY --from=rustdeps /opt/rust /opt/rust

RUN export DEBIAN_FRONTEND=noninteractive && apt-get update \
  && apt-get install -qy \
    wine \
    mold \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/*
