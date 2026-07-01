# OpenModelica build-deps Docker Images

[![Build, Release & Publish][badge-build-img]][workflow-build]

The Docker images used to build and deploy
[OpenModelica][openmodelica] with
[Jenkins][jenkins].

Images are published to:

- `ghcr.io/openmodelica/build-deps` (GitHub Container Registry)
- `docker.openmodelica.org/build-deps` (Nexus)

## Structure of the Repository

Every image lives on `main`, keyed by **operating system and OS version** rather
than by OpenModelica version. Each image is a **base** plus optional, layered
**add-ons**, so the heavy common tooling is built once and reused.

```text
main
├── apt/
│   └── Dockerfile          # multi-stage: all Ubuntu + Debian versions + add-ons
├── rpm/
│   └── Dockerfile          # planned: Fedora, AlmaLinux, Rocky Linux, RHEL
├── pacman/
│   └── Dockerfile          # placeholder (not implemented yet)
└── .ci/
    ├── matrix.yml          # source of truth: which images exist
    ├── matrix.py           # matrix.yml -> CI matrix / tag lookup
    ├── build.sh            # build one image (base + add-ons), write GHA cache
    └── publish.sh          # restore GHA cache, push + sign one image
```

- **Base image** — one per OS/OS-version. Contains everything needed to build
  OpenModelica (distro packages + common tooling: TeX, Qt, Python venv,
  ccache, …). This is what most CI jobs use.
- **Add-on image** — the base plus *one* thing the distro package manager can't
  provide or that needs a pinned version (e.g. CMake 4). Realized as an extra
  build **stage** (`FROM` the base stage) in the same Dockerfile, so shared
  layers are reused from cache.

Ubuntu and Debian share [apt/Dockerfile][apt-dockerfile]. The `DISTRO` and
`VERSION` build-args select the base image; the Qt package set is picked from
`${ID}:${VERSION_ID}` at build time. The base image is the `full` stage; each
add-on is a further stage (e.g. `--target cmake-4`).

Each image's `context`, `dockerfile`, `target`, `build_args` and `addons`
(add-on stage names) are declared in [.ci/matrix.yml][matrix-yml].

To add a new image, create or extend the OS's Dockerfile and list it in
[.ci/matrix.yml][matrix-yml] (a new version of an existing OS needs only a
matrix entry).

### Image naming & tags

One image repository per registry; OS, version and variant are encoded in the
**tag**:

| Tag                           | Mutable?  | Meaning                                               |
| ----------------------------- | --------- | ----------------------------------------------------- |
| `ubuntu-24.04`                | moving    | Latest base image for Ubuntu 24.04                    |
| `ubuntu-24.04-2.1.0`          | immutable | Pinned base, synthesized from git tag `v2.1.0`        |
| `ubuntu-24.04-cmake-4`        | moving    | Latest CMake 4 add-on on the 24.04 base               |
| `ubuntu-24.04-cmake-4-2.1.0`  | immutable | Pinned add-on, synthesized from git tag `v2.1.0`      |

Releasing is done by pushing a single repo-wide git tag `v<MAJOR>.<MINOR>.<PATCH>`
(e.g. `v2.1.0`). CI synthesizes the per-image immutable Docker tags from it and
publishes all images in one run. Day-to-day CI uses the **moving** tag; when an
OpenModelica release needs a frozen environment it pins the **immutable** tag.

### Currently provided images

| OS / version            | Base tag       | Add-ons            | Dockerfile          | Status      |
| ----------------------- | -------------- | ------------------ | ------------------- | ----------- |
| Ubuntu 26.04 (Resolute) | `ubuntu-26.04` | `rust`, `debug`    | `apt/Dockerfile`    | implemented |
| Ubuntu 24.04 (Noble)    | `ubuntu-24.04` | `cmake-4`, `debug` | `apt/Dockerfile`    | implemented |
| Ubuntu 22.04 (Jammy)    | `ubuntu-22.04` | `debug`            | `apt/Dockerfile`    | implemented |
| Debian 13 (Trixie)      | `debian-13`    | `cmake-4`, `debug` | `apt/Dockerfile`    | implemented |
| Debian 12 (Bookworm)    | `debian-12`    | `cmake-4`, `debug` | `apt/Dockerfile`    | implemented |
| Fedora, AlmaLinux, RHEL | `<os>-<ver>`   | –                  | `rpm/Dockerfile`    | planned     |
| Arch Linux (rolling)    | `arch-rolling` | –                  | `pacman/Dockerfile` | placeholder |

## Build locally

**Base image** — pick the distro and version with `DISTRO` and `VERSION`:

```bash
# Ubuntu
docker build --pull --no-cache \
  --target full \
  --build-arg DISTRO=ubuntu --build-arg VERSION=24.04 \
  --tag build-deps:ubuntu-24.04 \
  apt

# Debian
docker build --pull --no-cache \
  --target full \
  --build-arg DISTRO=debian --build-arg VERSION=13 --build-arg INTEL_OCL_PKGS= \
  --tag build-deps:debian-13 \
  apt
```

**Add-on image** — build the add-on's stage with `--target`. It reuses the
base's cached layers, so it only adds the extra step:

```bash
docker build --pull \
  --target cmake-4 \
  --build-arg DISTRO=ubuntu --build-arg VERSION=24.04 \
  --tag build-deps:ubuntu-24.04-cmake-4 \
  apt
```

> The values to pass (`context`, `--file`, `--target`, `--build-arg`) for any
> image are exactly its fields in [.ci/matrix.yml][matrix-yml].

## CI workflow

A single workflow, [build.yml][workflow-build-file], runs the whole pipeline so
that a release it creates can publish in the **same** run (a release created
with `GITHUB_TOKEN` cannot trigger a separate workflow):

```text
discover ─▶ build (all images, no push)
              └─▶ release (tag only) ─▶ publish-ghcr + publish-nexus
```

- **build** — on every push/PR to `main` (and as the gate before release),
  builds every base + add-on declared in `.ci/matrix.yml` (no push).
- **release** — on an repo-wide release tag, creates/updates the GitHub Release.
- **publish-ghcr / publish-nexus** — build and push (and on GHCR **sign**) the
  images to GHCR and Nexus. Moving tags (`<os>-<version>`) are updated on every
  push to `main`, the weekly schedule, and `workflow_dispatch`. Immutable tags
  (`<os>-<version>-<semver>`) are pushed on a release tag, and can also be republished via `workflow_dispatch` when a global `v<semver>` is supplied.

## Releasing a new image version

See **[RELEASING.md][releasing-md]** for the step-by-step process.

## License

The original Dockerfile was taken from
[OpenModelica/OpenModelicaBuildScripts][build-scripts].
See [LICENSE.md][license-md].

[badge-build-img]: https://github.com/OpenModelica/build-deps/actions/workflows/build.yml/badge.svg?branch=main
[workflow-build]: https://github.com/OpenModelica/build-deps/actions/workflows/build.yml
[openmodelica]: https://github.com/OpenModelica/OpenModelica
[jenkins]: https://test.openmodelica.org/jenkins/
[matrix-yml]: ./.ci/matrix.yml
[apt-dockerfile]: ./apt/Dockerfile
[workflow-build-file]: ./.github/workflows/build.yml
[releasing-md]: ./RELEASING.md
[build-scripts]: https://github.com/OpenModelica/OpenModelicaBuildScripts
[license-md]: ./LICENSE.md
