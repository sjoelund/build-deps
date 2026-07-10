# Multi-stage, shared image for the RPM-based distros: the enterprise family
# AlmaLinux / RHEL (all EL = Enterprise Linux, i.e. the `rhel`
# family) and Fedora. A single Dockerfile covers EL 9, 10 and Fedora 43, 44;
# DISTRO and VERSION are the only per-image differences, plus a handful of
# distro-specific packages (Qt5 vs Qt6, EPEL/CRB, autoconf 2.7x, gcc-toolset,
# omniORB) selected from ID/PLATFORM_ID at build time.
#
# The build-dependency list is the translation of the OpenModelica RPM spec
# (OpenModelicaBuildScripts/rpm/SPECS/openmodelica.spec.tpl, the `BuildRequires`)
# and the per-distro setup in that repo's Dockerfile.rpm-el{9,10}.template and
# Dockerfile.rpm-fc{42,43}.template (EPEL + CRB/powertools on EL only, autoconf
# 2.7x symlinks, the qmake symlink). It is kept explicit here — like
# apt/Dockerfile's package lists — instead of running `dnf builddep` against the
# OpenModelica repo, so the image is self-contained and does not depend on a
# matching source RPM being published.
#
# Stages / build targets:
#   base     OpenModelica build dependencies only
#   venv     Python virtualenv at /opt/venv, built in isolation and copied later
#   full     the published BASE image: base + common tooling + python venv
#
# Build the base image (the `full` target):
#   docker build --target full --build-arg DISTRO=almalinux --build-arg VERSION=10 \
#     --tag build-deps:almalinux-10 rpm
#   docker build --target full --build-arg DISTRO=almalinux --build-arg VERSION=9 \
#     --tag build-deps:almalinux-9 rpm
#   docker build --target full --build-arg DISTRO=fedora --build-arg VERSION=44 \
#     --tag build-deps:fedora-44 rpm
#
# RHEL: the public AlmaLinux images are bug-for-bug EL rebuilds, so the
# same recipe applies to a RHEL/UBI base — pass DISTRO/VERSION for a RHEL base
# image (e.g. redhat/ubi9) that your build host is entitled to.
#
# CI passes DISTRO, VERSION and the --target per image from .ci/matrix.yml.

ARG DISTRO=almalinux
ARG VERSION=10

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
    LANG=C.UTF-8 \
    LC_ALL=C.UTF-8

# Package lists as build-args, so they are easy to read and to override:
#   - BUILD_PKGS  : build dependencies common to every distro (spec's
#                   version-independent BuildRequires).
#   - QT6_PKGS    : Qt6 build set for EL 10 and Fedora. qt6-qtbase-devel already
#                   ships /usr/bin/qmake.
#   - QT5_PKGS    : Qt5 build set for EL 8 and EL 9 — this matches upstream's
#                   openmodelica.spec, which builds Qt5 on rhel <= 9 and reserves
#                   Qt6 for EL 10. On EL 9, EPEL's Qt6 stack is currently
#                   unbuildable anyway: qt6-linguist needs an LLVM (libclang) soname
#                   that EL 9 AppStream no longer ships, and LinguistTools is a
#                   required OMEdit Qt component. OMEdit's Quick3D animation backend
#                   (Qt6-only) is opt-in (OM_OMEDIT_ANIMATION_QUICK3D, OFF by
#                   default; OpenSceneGraph is the default), so Qt5 here is fine.
#   - FEDORA_PKGS : Fedora-only BuildRequires the EL builds skip — omniORB/lpsolve
#                   (the --with-omniORB branch, taken when rhel < 8, i.e. also on
#                   Fedora).
#   - OSG_PKGS    : OpenSceneGraph-devel, OMEdit's default 3D animation backend
#                   (Qt Quick 3D is the opt-in alternative). Added on every distro
#                   that packages it — EL 8/9 (from EPEL) and Fedora,
#                   EL 10 dropped OpenSceneGraph entirely, so it has no OSG
#                   backend and relies on Qt Quick 3D (Qt6) instead.
# Distro-specific extras handled in the RUN case block below:
#   - EL only : EPEL + CRB/powertools (Fedora needs neither).
#   - EL 8/9  : autoconf 2.7x (from EPEL) symlinked as autoconf/autoreconf/…,
#               Qt5 (QT5_PKGS) + qmake -> qmake-qt5, OpenSceneGraph-devel (OSG_PKGS)
#   - EL 8    : gcc-toolset-11 as well (default gcc 8.5 is too old)
#   - EL 10   : nothing extra (new enough autoconf, qt6 already in QT6_PKGS; no OSG)
#   - Fedora  : new enough autoconf/gcc; qmake ships with qt6-qtbase-devel;
#               OpenSceneGraph-devel (OSG_PKGS)
ARG BUILD_PKGS="\
    automake \
    bison \
    boost-devel \
    boost-static \
    clang \
    cmake \
    curl-devel \
    expat-devel \
    flex \
    gcc \
    gcc-c++ \
    gcc-gfortran \
    gettext \
    hdf5-devel \
    hwloc-devel \
    java-devel \
    lapack-devel \
    libffi-devel \
    libstdc++-static \
    libtool \
    libuuid-devel \
    make \
    readline-devel \
    tar \
    uuid \
    uuid-devel \
    xz"
# qtdeclarative provides the Qt Quick / Qml / QuickWidgets components; qtquick3d
# provides the Quick3D component of OMEdit's optional Qt Quick 3D animation
# backend (OM_OMEDIT_ANIMATION_QUICK3D, OFF by default). Both sets carry
# qtdeclarative; only Qt6 carries qtquick3d (Qt5 has no qtquick3d on EL/Fedora),
# so the Quick3D backend is only available on the Qt6 images (EL 10 / Fedora).
ARG QT5_PKGS="\
    qt5-linguist \
    qt5-qt3d-devel \
    qt5-qtbase-devel \
    qt5-qtdeclarative-devel \
    qt5-qtsvg-devel \
    qt5-qttools \
    qt5-qtwebkit-devel \
    qt5-qtxmlpatterns-devel"
ARG QT6_PKGS="\
    qt6-linguist \
    qt6-qt3d-devel \
    qt6-qt5compat-devel \
    qt6-qtbase-devel \
    qt6-qtdeclarative-devel \
    qt6-qthttpserver-devel \
    qt6-qtquick3d-devel \
    qt6-qtsvg-devel \
    qt6-qttools-devel \
    qt6-qtwebengine-devel"
ARG FEDORA_PKGS="\
    lpsolve-devel \
    omniORB-devel"
ARG OSG_PKGS="OpenSceneGraph-devel"

RUN dnf install -y dnf-plugins-core \
  && . /etc/os-release \
  # Fedora is matched by ID (its os-release leaves PLATFORM_ID empty); the EL
  # family (almalinux/rhel) is matched by PLATFORM_ID (platform:elN).
  && if [ "${ID}" = "fedora" ]; then \
       case "${VERSION_ID}" in \
         43 | 44) qt="${QT6_PKGS}" && extra="${FEDORA_PKGS} ${OSG_PKGS}" ;; \
         *) echo "Unsupported Fedora VERSION_ID: ${VERSION_ID}" >&2 && exit 1 ;; \
       esac; \
     else \
       case "${PLATFORM_ID}" in \
         platform:el8) \
           dnf install -y epel-release \
           && dnf config-manager --set-enabled powertools \
           && qt="${QT5_PKGS}" \
           && extra="autoconf2.7x gcc-toolset-11-gcc gcc-toolset-11-gcc-c++ gcc-toolset-11-gcc-gfortran ${OSG_PKGS}" ;; \
         platform:el9) \
           dnf install -y epel-release \
           && dnf config-manager --set-enabled crb \
           && qt="${QT5_PKGS}" \
           && extra="autoconf2.7x ${OSG_PKGS}" ;; \
         platform:el10) \
           dnf install -y epel-release \
           && dnf config-manager --set-enabled crb \
           && qt="${QT6_PKGS}" \
           # EL 10 has no OpenSceneGraph package; OMEdit uses Qt Quick 3D (Qt6).
           && extra="" ;; \
         *) echo "Unsupported PLATFORM_ID: ${PLATFORM_ID}" >&2 && exit 1 ;; \
       esac; \
     fi \
  && dnf -y upgrade \
  && dnf install -y ${BUILD_PKGS} ${qt} ${extra} \
  # EL 8/9 ship autoconf 2.69; OpenModelica's autoreconf needs 2.7x (EPEL's
  # autoconf2.7x installs autoconf27/autoreconf27/…). Expose them unversioned.
  && if command -v autoconf27 >/dev/null 2>&1; then \
       ln -sf "$(command -v autoconf27)"   /usr/local/bin/autoconf; \
       ln -sf "$(command -v autoreconf27)" /usr/local/bin/autoreconf; \
       ln -sf "$(command -v autoheader27)" /usr/local/bin/autoheader; \
       ln -sf "$(command -v autom4te27)"   /usr/local/bin/autom4te; \
     fi \
  # OMEdit's qmake-based pieces expect a plain `qmake`; EL 8/9 provide qmake-qt5.
  && if [ ! -e /usr/bin/qmake ] && [ -e /usr/bin/qmake-qt5 ]; then \
       ln -s /usr/bin/qmake-qt5 /usr/bin/qmake; \
     fi \
  && dnf clean all \
  && rm -rf /var/cache/dnf

# CI mounts fresh named volumes under /cache; Docker seeds an empty volume with
# the mount point's image permissions, so create them world-writable for the
# non-root Jenkins uid (sccache/runtest/omlibrary caches).
RUN mkdir -p /cache/sccache /cache/runtest /cache/omlibrary \
  && chmod -R 0777 /cache

# ── venv: Python tooling, isolated so it caches independently ─────────────────
# Built once and copied into `full`. Pins python3.12 explicitly (EL 9's default
# python3 is 3.9, too old for current fmpy; python3.12 is packaged on EL 8/9/10
# and Fedora), so `full` installs python3.12 as well and the relocated /opt/venv
# works there unchanged regardless of the distro's default python3.
FROM base AS venv
# python3.12 bundles ensurepip, so `python3.12 -m venv` bootstraps pip without a
# separate pip package (EL's python3.12-pip name does not exist on Fedora).
RUN dnf install -y python3.12 \
  && dnf clean all \
  && rm -rf /var/cache/dnf
RUN python3.12 -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
RUN pip install --no-cache-dir --upgrade pip
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
#   - common tooling shared by all versions (git, ccache, doc tools, …)
#   - the Python venv built in the `venv` stage
FROM base AS full

# COMMON_PKGS: tooling on top of the raw build dependencies (fetching sources,
# faster rebuilds, the User's Guide / Doxygen toolchain). All from BaseOS/
# AppStream/CRB/EPEL, which `base` already enabled.
ARG COMMON_PKGS="\
    ccache \
    diffutils \
    doxygen \
    git \
    graphviz \
    patch \
    python3.12 \
    unzip \
    wget \
    which \
    zip"
RUN dnf install -y ${COMMON_PKGS} \
  && dnf clean all \
  && rm -rf /var/cache/dnf

# Pull in the venv built in its own stage instead of rebuilding it here.
COPY --from=venv /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
