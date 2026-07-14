# chatmail dovecot (downstream)

This is [chatmail](https://github.com/chatmail)'s downstream rebuild of Debian's
`dovecot` source package (upstream 2.3.21), used by delta.chat mail servers. It
tracks Debian's packaging closely and only adds what the chatmail deployment
needs; the `debian/` tree and build behaviour are meant to stay faithful to the
archive so the binaries match what Debian would produce.

## Continuous integration

The GitHub Actions workflows under `.github/workflows/` deliberately reproduce
how Debian **builds and gates** this package rather than inventing a parallel
CI. Each piece maps to an upstream Debian definition or tool:

| Our CI step | What it mirrors | Upstream definition / docs |
|---|---|---|
| `build` job — `gbp buildpackage` as an unprivileged fakeroot user, build-deps from `debian/control` | how a Debian buildd builds the source package | git-buildpackage manual — <https://gbp.sigxcpu.org/manual/> |
| whole pipeline shape (build → autopkgtest → reprotest) | the salsa-ci pipeline recipe our `debian/salsa-ci.yml` `include:`s | salsa-ci pipeline — <https://salsa.debian.org/salsa-ci-team/pipeline> (recipe: [`recipes/debian.yml`](https://salsa.debian.org/salsa-ci-team/pipeline/-/raw/master/recipes/debian.yml)) |
| `autopkgtest` job — `debian/tests` in Incus system containers | `ci.debian.net` / debci running DEP-8 tests in systemd system containers | debci — <https://ci.debian.net/> ; autopkgtest — <https://salsa.debian.org/ci-team/autopkgtest> |
| testbed build via `autopkgtest-build-incus` | the `autopkgtest-build-lxd` / `-incus` tool (its alias format drives our image lookup) | [`tools/autopkgtest-build-lxd`](https://salsa.debian.org/ci-team/autopkgtest/-/raw/master/tools/autopkgtest-build-lxd) |
| `reprotest` job — `--vary` set, run once per distro | the salsa-ci reprotest job | reprotest — <https://salsa.debian.org/reproducible-builds/reprotest> ([manpage](https://manpages.debian.org/unstable/reprotest/reprotest.1.en.html)) ; Reproducible Builds — <https://reproducible-builds.org/> |
| Incus system containers (chatmail house standard) | — | Incus — <https://linuxcontainers.org/incus/docs/> ; cmlxc — <https://github.com/chatmail/cmlxc> |

Package overview and Debian's own build history: the
[Debian dovecot packaging](https://salsa.debian.org/debian/dovecot) and the
[package tracker](https://tracker.debian.org/pkg/dovecot).

### reprotest: disabled upstream, re-enabled here

Debian **disables** reprotest for dovecot. See `debian/salsa-ci.yml`, inherited
verbatim from the Debian packaging:

```yaml
# The test suite does not pass reprotest
variables:
  SALSA_CI_DISABLE_REPROTEST: 1
```

That failure was the *test suite* running (as root) under reprotest's
build-path/user variations. We build with `DEB_BUILD_OPTIONS=nocheck` (tests
don't affect the shipped artifacts) as an unprivileged user, under which the
reproducibility check passes — so we re-enable it. The variations we toggle off
mirror the salsa-ci defaults.

## Workflow layout

- **`ci`** (`build-staging-deb.yml`) — runs on PRs and `claude/**` branches;
  fans out a bookworm/trixie × amd64/arm64 matrix, each leg calling the reusable
  pipeline. Checks read as `‹distro›/‹arch› / ‹job›`.
- **`deb pipeline`** (`pipeline.yml`) — reusable (`workflow_call`); one
  distro/arch leg: `build → autopkgtest → reprotest` (the latter two on amd64,
  since they are arch-independent).
- **`release`** (`build-deb.yml`) — runs on `master` and `upstream/*` tags;
  builds and publishes the `.deb`s (and `.buildinfo`) to download.delta.chat and
  GitHub Releases.

The Debian version carries a downstream suffix (`…+chatmailN`); installs are
dpkg-pinned, so version ordering relative to Debian's own revisions is
irrelevant.
