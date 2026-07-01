# Releasing a new image version

A "release" publishes **every image** — each base image and *all* of its
add-ons — under both a moving and an immutable tag, to GHCR and Nexus.
Releases are driven entirely by **git tags**; you never push images by hand.

> TL;DR — merge your change to `main`, then push a git tag
> `v<semver>` (e.g. `v2.1.0`). CI does the rest.

## The tag grammar

```text
<os>-<os-version>-<semver>
```

| Part           | Example                  | Notes                                                                                         |
| -------------- | ------------------------ | --------------------------------------------------------------------------------------------- |
| `<os>`         | `ubuntu`                 | A directory at the repo root holding `<os>/Dockerfile`.                                       |
| `<os-version>` | `24.04`, `13`, `rolling` | Selects the version (a build-arg) within that OS.                                             |
| `<semver>`     | `2.1.0`                  | **This repository's** version (the recipe), independent of OpenModelica. `MAJOR.MINOR.PATCH`. |

The pair `<os>-<os-version>` must match an entry in
[.ci/matrix.yml][ci-matrix] (you can list valid prefixes with
`python .ci/matrix.py all`).

### What gets published

For git tag `v2.1.0` the publish workflows build and push:

| Image            | Moving tag             | Immutable tag                |
| ---------------- | ---------------------- | ---------------------------- |
| base             | `ubuntu-24.04`         | `ubuntu-24.04-2.1.0`         |
| add-on `cmake-4` | `ubuntu-24.04-cmake-4` | `ubuntu-24.04-cmake-4-2.1.0` |

to both `ghcr.io/openmodelica/build-deps` (signed with cosign) and
`docker.openmodelica.org/build-deps`. Add-ons are extra stages that build
`FROM` the base stage in the same Dockerfile, so a release is internally
consistent and shared layers come from the build cache.

## Choosing the next semver

Bump relative to the last release tag (`git tag --list 'v*'`):

- **PATCH** (`2.1.0 → 2.1.1`) — rebuild for upstream package updates / security
  fixes, no intended behavior change.
- **MINOR** (`2.1.0 → 2.2.0`) — added a tool or an add-on, backward compatible.
- **MAJOR** (`2.1.0 → 3.0.0`) — removed/renamed something consumers rely on, or
  a base OS bump that changes the toolchain.

One git tag releases **all** images at once; there is a single shared semver
for the whole repository.

## Step by step

1. **Edit the Dockerfile** for the image: each OS has a single multi-stage
   `<os>/Dockerfile` (e.g. all Ubuntu versions share `ubuntu/Dockerfile`) where
   the base is the `full` stage and add-ons are extra stages such as `cmake-4`.
   Its `context`/`dockerfile`/`target`/`build_args` are in
   [.ci/matrix.yml][ci-matrix].

2. **Open a PR to `main`.** [build.yml][build-yml] builds
   every image (no push). Confirm your image builds. Build it locally too —
   reuse the `context` / `dockerfile` / `target` / `build_args` from
   `.ci/matrix.yml`:

   ```bash
   # Ubuntu base (the `full` stage):
   docker build --pull --target full --build-arg UBUNTU_VERSION=24.04 \
     --tag build-deps:ubuntu-24.04 ubuntu
   # each add-on (its own --target stage, same Dockerfile):
   docker build --pull --target cmake-4 --build-arg UBUNTU_VERSION=24.04 \
     --tag build-deps:ubuntu-24.04-cmake-4 ubuntu
   ```

3. **Merge to `main`.**

4. **Tag and push** the release from the merge commit on `main`:

   ```bash
   git checkout main && git pull
   git tag v2.1.0
   git push origin v2.1.0
   ```

5. **CI publishes automatically.** Pushing the tag runs the single
   [build.yml](./.github/workflows/build.yml) pipeline, which in one run:
   builds the images, creates/updates the GitHub Release, then builds + pushes
   the tagged image (base + add-ons) to GHCR (signed) and Nexus.

   Watch the Actions tab; when green the new tags are live.

## Re-running a publish without a new tag

Use the **workflow_dispatch** trigger on
[build.yml](./.github/workflows/build.yml):

- **Leave the tag field empty** — rebuilds and re-pushes all moving tags
  (`<os>-<version>`). Handy after a transient registry failure or to force a
  fresh rebuild without cutting a new release.
- **Pass a global tag** (e.g. `v2.1.0`) — re-publishes every image with both
  the moving and the immutable tags for that version (the release step is
  skipped).
- **Pass a single image tag** (e.g. `ubuntu-24.04-2.1.0`) — re-publishes
  only that one image with its moving and immutable tags.

## Adding a brand-new image

1. For a **new OS**, create the single multi-stage `<os>/Dockerfile` (see
   `ubuntu/Dockerfile`). For a **new version of an existing OS**, no new
   Dockerfile is needed — just add a matrix entry: e.g. a new Ubuntu version
   adds `dockerfile: ubuntu/Dockerfile`, `target: full` and
   `build_args: { UBUNTU_VERSION: "<ver>" }`.
2. Add the image to [.ci/matrix.yml](./.ci/matrix.yml).
3. PR → merge → release as above with `v1.0.0`.

## Adding an add-on

Add-ons are extra build **stages** in the image's Dockerfile that build `FROM`
the base stage and add exactly one thing:

```dockerfile
FROM full AS my-addon          # `full` is the base stage
# ... install the one extra thing (a pinned toolchain, a source-built lib) ...
```

Then list the stage name under the image's `addons:` in
[.ci/matrix.yml](./.ci/matrix.yml). CI builds it with `--target my-addon` and
publishes `…-my-addon` (moving) and `…-my-addon-<semver>` (immutable). For
single-stage bases (Debian/Arch) add the `full`/base stage name and a `target:`
to the entry first, or split that Dockerfile into stages the same way.

## Arch / rolling snapshots

Arch has no version number. The moving tag `arch-rolling` always tracks the
latest build. For a reproducible pin, release with a date-stamped tag,
e.g. `arch-rolling-2026.6.1` — the publish workflows match a `\d+\.\d+\.\d+`
suffix, so use `YYYY.M.D` (without padding, e.g. `2026.6.1` not `2026.06.01`):
SemVer 2.0.0 forbids leading zeros in numeric identifiers.

[ci-matrix]: ./.ci/matrix.yml
[build-yml]: ./.github/workflows/build.yml
