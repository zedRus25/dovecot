# Handover: Debian-style CI (unit tests, autopkgtest, reproducibility)

Status as of 2026-07-13. Branch: `claude/adoring-maxwell-c221jh`
(commits `0dcb4279` and `5366511e` on top of `master`).

## Background / goal

This repo is a fork of Debian's dovecot packaging
(https://salsa.debian.org/debian/dovecot/) for chatmail, pinned to the last
2.3-era Debian upload (`1:2.3.21+dfsg1-3`, upstream 2.3.21) and therefore
diverged from debian-security. The goals of this work:

1. Run the same testing Debian's CI runs (build-time unit tests + the
   `debian/tests/` autopkgtest suite) in our GitHub Actions.
2. Mimic Debian's build process closely enough that the build is
   reproducible, and verify that in CI.

Backporting the debian-security changes is explicitly **out of scope** here
(future work; see "Next steps").

## What changed

### `.github/workflows/build-staging-deb.yml` (PR / feature-branch CI)

- Triggers: `pull_request`, `workflow_dispatch`, and now also `push` to
  `claude/**` branches (added so feature branches exercise CI without a PR —
  feel free to remove or widen that glob once the PR flow is the norm).
- `staging` job (matrix: bookworm/trixie x amd64/arm64):
  - Build deps come from `apt-get build-dep ./dovecot`, i.e. resolved from
    `debian/control` like a buildd would, replacing the hand-maintained
    package list.
  - **Unit tests are enabled** (`DEB_BUILD_OPTIONS=nocheck` removed):
    `dh_auto_test` runs `make check` during the build.
  - The build runs as an unprivileged `builder` user (fakeroot for binary
    targets, which is what `Rules-Requires-Root: binary-targets` expects).
    **This is load-bearing**: the upstream test `test-buffer-istream`
    (`buffer_append_full_file`) chmods a file to 0000 and asserts the read
    fails — as root the read succeeds and the test FAILS. This is also the
    real reason Debian disabled reprotest for dovecot on salsa (commit
    `6bd4255` in the salsa packaging repo, 2021-01-04).
  - The whole `build-area/` (debs, dsc, tarballs, `.buildinfo`) is uploaded
    as a GitHub artifact `build-area-<distro>-<arch>` for the downstream
    jobs, in addition to the existing rsync upload to download.delta.chat.
- `autopkgtest` job (same matrix, plain VMs — **not** `container:`):
  - Runs the `debian/tests/control` suite against the built debs, mimicking
    ci.debian.net: that service uses lxc system containers with systemd as
    PID 1; our equivalent is `autopkgtest-build-podman --init systemd` plus
    the `podman` virt-server.
  - One pristine testbed per test (`--test-name`, four invocations):
    `doveadm`, `systemd`, `command1`, `testmails`. **`command1` is the
    autopkgtest-assigned name** of the anonymous `Test-Command:` stanza in
    `debian/tests/control` (the `debian/tests/usage` run-parts scripts).
  - `--ignore-restrictions=breaks-testbed` is required because the podman
    runner cannot revert testbeds; it is safe because each test gets a fresh
    container. Fresh-per-test also matters for ordering: `usage` rewrites
    `/etc/dovecot`, so running `testmails` after it in the same testbed
    would fail.
  - Why not the null runner inside the build container: GitHub `container:`
    jobs cannot run systemd (the runner owns PID 1), and the `systemd` and
    `usage` tests call `systemctl` against a live systemd.
- `reprotest` job (bookworm + trixie, amd64):
  - The salsa-ci job Debian skips for this package — viable for us because
    the only documented failure was the run-as-root unit test, and we run
    reprotest as non-root **with `DEB_BUILD_OPTIONS=nocheck`** (the test
    suite does not affect the shipped artifacts).
  - Builds the `.dsc` from the staging artifact twice under variations and
    compares. Disabled variations, mirroring salsa-ci defaults plus
    container constraints: `-time` (faketime is flaky), `-build_path`
    (buildds use a fixed path anyway), `-user_group`/`-domain_host` (need
    sudo setup), `-fileordering` (disorderfs needs FUSE, unavailable in
    unprivileged containers), `-aslr` (blocked by the default docker seccomp
    profile). Still varied: environment, hostname-independent env, kernel
    uname, locales, exec path, timezone, umask, num_cpus.
  - On failure it runs diffoscope and uploads the two build trees as an
    artifact.

### `.github/workflows/build-deb.yml` (master pushes + `upstream/*` tags)

- Same buildd-style changes: build-deps from `debian/control`, non-root
  `builder` user, **unit tests enabled**.
- Release uploads now include the **`.buildinfo`** files (renamed
  `dovecot_<ver>_<arch>_<distro>.buildinfo`): they record the exact
  toolchain and build-dep versions, which is what lets anyone reproduce and
  verify the published binaries later (debrebuild-style). The rsync upload
  always shipped the whole `build-area/`, so it already included them.

## Validation done

- Full `dpkg-buildpackage` (tests enabled) was run in a Linux sandbox
  (Ubuntu 24.04):
  - As **root**: fails exactly on `test-buffer-istream` — confirms the
    non-root design.
  - As **non-root**: entire suite passes (113 test dirs). The only 3
    failures observed were sandbox artifacts that do not exist on GitHub
    runners: an egress MITM proxy (broke an http-client timeout test with
    `403 Forbidden`), odd IPv6 loopback behavior (`test_program_refused`),
    and a >108-char build path overflowing `sun_path`
    (`test-imap-client-hibernate`). Debian's own buildds ran this suite
    green for 2.3.21+dfsg1-3.
  - Note: pigeonhole's own `make check` is **not** run — same as in
    Debian's build (`debian/rules` has no test hook for the pigeonhole
    subdir). Intentional: mimic, don't extend.
- YAML validated; **first real pipeline run is in flight**:
  https://github.com/zedRus25/dovecot/actions/runs/29205496443
  The `autopkgtest` (podman+systemd bootstrap, esp. on the arm64 runners)
  and `reprotest` jobs are research-validated but had no prior Actions run —
  expect at most flag-level fixes there.

## Reproducibility facts worth knowing

- Dovecot 2.3 was reproducible in Debian. Its one historical issue — the
  build path captured in `dovecot-dev`'s `dovecot-config` — is already
  handled in `debian/rules` (sed strips `-ffile-prefix-map`/
  `-fdebug-prefix-map` from that file).
- `SOURCE_DATE_EPOCH` comes automatically from the top `debian/changelog`
  entry via `dpkg-buildpackage`; dpkg's default `fixfilepath` feature makes
  compiled output build-path-independent.
- Debian overall is **not** 100% reproducible: ~97% of trixie/amd64 is
  bit-for-bit reproduced by the semi-official rebuilderd infrastructure
  (https://reproduce.debian.net/), and since May 2026 reproducibility
  regressions block testing migration for Debian 14. If you ever want to
  compare against Debian's official 2.3.19 (bookworm) debs, rebuild from
  their `.buildinfo` with `debrebuild` rather than rolling your own
  environment.

## Lessons from the first pipeline run (run #1, 2026-07-12)

Two failures, both fixed in commit `525ca0fd`:

- **This fork has no git tags** (GitHub forks don't copy them), so gbp
  could not build the orig tarball from `upstream/2.3.21+dfsg1`
  (`gbp:error: ... is not a valid treeish`). The workflows now fall back
  to `--git-upstream-tree=SLOPPY` (orig tarball = branch minus `debian/`)
  when the tag is absent. Caveats: the generated orig tarball is not
  byte-identical to Debian's dfsg tarball, and the `upstream/*` tag
  trigger of build-deb.yml only fires once tags are actually pushed to
  this repo — consider `git push origin 'refs/tags/upstream/*'` from a
  clone that has them (e.g. from salsa or the chatmail repo), which also
  makes gbp use the proper tag again.
- **`apt-get install` hung for 6 h on debian:12**: git-buildpackage's
  recommends pull in pbuilder, whose setup blocks without a preseeded
  mirror — this is why the pre-rewrite workflow wrote
  `MIRRORSITE=... > /etc/pbuilderrc`. Fixed properly with
  `--no-install-recommends` (buildd-faithful anyway) +
  `DEBIAN_FRONTEND=noninteractive`, plus `timeout-minutes` on every job
  and a concurrency group cancelling superseded runs.

## Next steps

1. **Watch the current pipeline run** and fix what surfaces. Remaining
   likely spots: podman/systemd testbed on `ubuntu-24.04-arm`, reprotest
   variation quirks, autopkgtest exit code 2 (skips) semantics.
2. Once green, consider making the `autopkgtest` + `reprotest` jobs
   required checks for PRs.
3. Security backport (the original motivation for this fork's divergence):
   upstream 2.3.21.1 is the final 2.3 release and contains the fixes for
   CVE-2024-23184 / CVE-2024-23185; Debian's last 2.3 upload to sid was
   `1:2.3.21.1+dfsg1-1` (2025-02-11) before sid moved to 2.4. Rebasing this
   repo onto 2.3.21.1 (or cherry-picking those fixes plus any
   debian-security patches for bookworm's 2.3.19) is the natural next task,
   and the new CI is exactly the safety net for it.
4. Optional salsa-ci parity extensions, in rough order of value: `lintian`,
   `piuparts`, `blhc` — all cheap jobs consuming the existing build
   artifacts.

## Operational notes

- Manual run: Actions tab -> "staging" -> Run workflow -> pick the branch
  (works now that the workflow is registered).
- Debugging a failed autopkgtest job: download the
  `autopkgtest-logs-<distro>-<arch>` artifact; each test's directory
  contains the full testbed log.
- Debugging a reprotest failure: the job prints diffoscope text output and
  uploads `reprotest-<distro>` with both build trees.
- The unit-test suite runs inside `gbp buildpackage`; to reproduce a test
  failure locally:
  `apt-get build-dep ./ && dpkg-buildpackage -us -uc -b` **as a non-root
  user** (as root, `test-buffer-istream` fails by design).
