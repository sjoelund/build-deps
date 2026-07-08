# Multi-stage, shared image for Alpine Linux.
#
# A single Dockerfile covers the supported Alpine versions; VERSION is the only
# per-image difference. Alpine has no OpenModelica apk repository, so the build
# dependencies are installed directly with apk (there is no `apt build-dep`
# equivalent). Package names were resolved against the Alpine main/community
# repositories.
#
# Stages / build targets:
#   base        OpenModelica build dependencies (toolchain + core libraries)
#   venv        Python virtualenv at /opt/venv, built in isolation and copied later
#   full        the published BASE image: base + Qt6 + OpenSceneGraph + python venv
#   omsimulator ADD-ON: full + X11/mesa dev libs for the OMSimulator build
#
# Build the base image (the `full` target):
#   docker build --target full --build-arg VERSION=3.22 \
#     --tag build-deps:alpine-3.22 apk
#
# Build the OMSimulator add-on image (the `omsimulator` target):
#   docker build --target omsimulator --build-arg VERSION=3.22 \
#     --tag build-deps:alpine-3.22-omsimulator apk
#
# CI passes VERSION and the --target per image from .ci/matrix.yml.

ARG VERSION=3.22

# ── base: OpenModelica build dependencies only ────────────────────────────────
FROM alpine:${VERSION} AS base
ARG VERSION

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
    LANG=C.UTF-8 \
    LC_ALL=C.UTF-8

# Qt6, Boost, LAPACK/BLAS, HDF5 and OpenSceneGraph live in the community repo,
# so make sure both main and community are enabled and pinned to this version.
RUN printf '%s\n' \
      "https://dl-cdn.alpinelinux.org/alpine/v${VERSION}/main" \
      "https://dl-cdn.alpinelinux.org/alpine/v${VERSION}/community" \
      > /etc/apk/repositories \
  && apk update && apk upgrade

# Core toolchain and libraries needed to build the OpenModelica compiler.
# Mirrors OMCompiler/README.Linux.md §1.2 (autotools/cmake, g++/gfortran, clang,
# LAPACK/BLAS, hdf5, hwloc, boost, libcurl, expat, ncurses/readline, …).
RUN apk add --no-cache \
    autoconf \
    automake \
    bash \
    bison \
    boost-dev \
    build-base \
    bzip2-dev \
    ccache \
    clang \
    cmake \
    coreutils \
    curl \
    curl-dev \
    expat-dev \
    flex \
    g++ \
    gfortran \
    git \
    hdf5-dev \
    hwloc-dev \
    lapack-dev \
    libtool \
    libxml2-dev \
    libxslt-dev \
    linux-headers \
    make \
    musl-dev \
    ncurses-dev \
    openblas-dev \
    openjdk17-jre-headless \
    openmp-dev \
    openssl-dev \
    patch \
    pkgconf \
    readline-dev \
    tar \
    unzip \
    util-linux \
    wget \
    xz \
    zip \
    zlib-dev

# CI mounts fresh named volumes under /cache; Docker seeds an empty volume with
# the mount point's image permissions, so create them world-writable for the
# non-root Jenkins uid (emscripten/sccache/runtest/omlibrary caches).
RUN mkdir -p /cache/emscripten /cache/sccache /cache/runtest /cache/omlibrary \
  && chmod -R 0777 /cache

# ── venv: Python tooling, isolated so it caches independently ─────────────────
# Built once and copied into `full`. Uses the same python3 as `full` (same
# base image), so the relocated /opt/venv works there unchanged.
FROM base AS venv
RUN apk add --no-cache python3 py3-pip
RUN python3 -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
# Use permalink for doc/UsersGuide/source/requirements.txt to keep builds
# deterministic.
RUN pip install --no-cache-dir \
    fmpy==0.3.29 \
    junit_xml \
    lxml \
    ompython==3.6.0 \
    PyGithub \
    simplejson \
    svgwrite \
  && pip install --no-cache-dir -r \
    https://raw.githubusercontent.com/OpenModelica/OpenModelica/9c0dc9a8ab50ba652109584cb3fecaef86640b66/doc/UsersGuide/source/requirements.txt

# ── full: the published base image ────────────────────────────────────────────
#   - Qt6 for OMEdit (base + the modules OpenModelica links against)
#   - OpenSceneGraph for the OMEdit 3D animation view
#   - doc generators (doxygen, graphviz) and a few build utilities
FROM base AS full

# Package lists as build-args, so they are easy to read and to override.
#   - QT_PKGS   : Qt6 modules OMEdit builds against
#   - EXTRA_PKGS: OpenSceneGraph, doc generators and misc build utilities
ARG QT_PKGS="\
    qt6-qt5compat-dev \
    qt6-qtbase-dev \
    qt6-qtdeclarative-dev \
    qt6-qthttpserver-dev \
    qt6-qtquick3d-dev \
    qt6-qtscxml-dev \
    qt6-qtserialport-dev \
    qt6-qtshadertools-dev \
    qt6-qtsvg-dev \
    qt6-qttools-dev \
    qt6-qtwebengine-dev \
    qt6-qtwebsockets-dev"
ARG EXTRA_PKGS="\
    doxygen \
    gnuplot \
    graphviz \
    jq \
    openscenegraph-dev"
RUN apk add --no-cache ${QT_PKGS} ${EXTRA_PKGS}

# Pull in the venv built in its own stage instead of rebuilding it here.
COPY --from=venv /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# ── OMSimulator add-on: full + X11/mesa dev libs OMSimulator links against ────
# Mirrors OMSimulator/.CI/alpine/Dockerfile plus the GLFW/GUI X11 dependencies.
# Everything else OMSimulator needs (automake, cmake, g++, git, libtool,
# linux-headers, make, musl-dev, readline-dev, util-linux) is already in `base`,
# so only the X11/mesa dev libraries are added here. Built with
# `--target omsimulator`.
FROM full AS omsimulator
LABEL org.opencontainers.image.description="OMSimulator build-deps"
RUN apk add --no-cache \
    libxcursor-dev \
    libxi-dev \
    libxinerama-dev \
    libxrandr-dev \
    mesa-dev
