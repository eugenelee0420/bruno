# Tasks

## 1. Remove existing automation

- [x] 1.1 Delete `.github/workflows/tests-linux.yml`, `tests-macos.yml`, `tests-windows.yml`, `lint-checks.yml`, `benchmarks.yml`, `flaky-test-detector.yml`, and `npm-bru-cli.yml`; verify `ls .github/workflows` is empty before adding the new file and that `git status` shows exactly seven deletions.
- [x] 1.2 Check `contributing.md` and `docs/` for references to the deleted workflows and update them in the same commit; verify with `grep -rn "tests-linux\|tests-windows\|tests-macos\|lint-checks\|benchmarks.yml\|flaky-test-detector\|npm-bru-cli" --include="*.md" .` returning only intentional historical references.

## 2. Add the trigger and validation job

- [x] 2.1 Create `.github/workflows/release.yml` with `on.push.branches: ['release/v*']`, `concurrency.group: release-${{ github.ref_name }}`, `cancel-in-progress: false`, and default `permissions: contents: read`; verify the file parses with `actionlint` (or `yamllint`) and the trigger matches only `release/v*` branches.
- [x] 2.2 Implement the `validate` job: derive `TAG` from `GITHUB_REF_NAME` by stripping `release/`, derive `VERSION` by stripping `v`, reject tags not matching `^v[0-9]+\.[0-9]+\.[0-9]+(-[0-9A-Za-z.-]+)?$`, and expose `tag`/`version` as job outputs; verify a local shell reproduction of the parsing accepts `release/v4.4.0` and rejects `release/test`.
- [x] 2.3 Add duplicate checks to `validate`: `gh release view "$TAG"` and `git ls-remote --tags origin "refs/tags/$TAG"` must both report absence; verify by running the job on a branch whose tag already exists and confirming it fails in under a minute without starting a build.

## 3. Build Linux artifacts

- [x] 3.1 Add the `build-linux` job on `ubuntu-24.04` that uses `./.github/actions/common/setup-node-deps`, runs `npm pkg set version="$VERSION" --workspace=packages/bruno-electron`, `npm run build:web`, then `npm run build:electron:linux`; verify the job fails if any of those commands fail and that the version rewrite is visible in `packages/bruno-electron/package.json` inside the runner.
- [x] 3.2 Add a verification step asserting `@lydell/node-pty-linux-x64` exists in `node_modules` and failing the job if it does not; verify the step passes on a clean Linux x64 runner.
- [x] 3.3 Add a verification step asserting exactly one installer exists (`*x86_64*.AppImage` whose filename contains `$VERSION`) and that no `*.deb` or `*.rpm` files are present; verify the step fails on a deliberately renamed fixture file.
- [x] 3.4 Upload that AppImage as the `linux-installers` Actions artifact with `if-no-files-found: error`; verify the artifact listing contains one file and no `linux-unpacked` directory or `builder-effective-config.yaml`.

## 4. Build Windows artifacts

- [x] 4.1 Add the `build-windows` job on `ubuntu-24.04` with the Wine install step (`dpkg --add-architecture i386`, `apt-get update`, `apt-get install -y --no-install-recommends wine wine64 wine32:i386`); verify `command -v wine` and `wine --version` succeed in the job before the build step. The first run failed because `wine64`/`wine32:i386` alone do not provide `/usr/bin/wine`; re-run to confirm the fix.
- [x] 4.2 In that job, run the composite setup, `npm pkg set version="$VERSION" --workspace=packages/bruno-electron`, force-install `@lydell/node-pty-win32-x64@1.1.0` and `@lydell/node-pty-win32-arm64@1.1.0` with `--no-save --force`, run `npm run build:web`, then `npm run build:electron:win`; verify the job produces `out/*_x64_win.exe` and `out/*_arm64_win.exe` whose names contain `$VERSION` and that both `@lydell/node-pty-win32-*` directories are present.
- [x] 4.3 Upload the installers (and `*.exe.blockmap` files) as the `windows-installers` Actions artifact with `if-no-files-found: error`; verify the artifact listing contains the x64, arm64, and combined `.exe` files and no `win-unpacked` directory.

## 5. Publish the release

- [x] 5.1 Add the `publish` job that `needs` both build jobs, runs on `ubuntu-24.04`, declares `permissions: contents: write`, downloads both Actions artifacts into one directory, re-checks that the release does not exist, and runs `gh release create "$TAG" --target "$GITHUB_REF_NAME" --title "$TAG" --notes "See https://github.com/usebruno/bruno/releases/tag/$TAG" <files>`; verify with `gh release view "$TAG" --json isDraft,targetCommitish,assets` that it is live, targets `release/vX.Y.Z`, and lists the AppImage, the x64/arm64/combined `.exe` installers, and their `.blockmap` files.
- [x] 5.2 Verify duplicate protection end to end: push another commit to the same released branch, confirm the run fails at `validate`, and confirm via `gh release view --json assets` that asset IDs and sizes are unchanged.
- [x] 5.3 Verify failure atomicity: introduce a temporary build failure on a test branch (for example `VERSION=bad` or a failing renderer build), confirm no tag and no release are created with `git ls-remote --tags` and `gh release view`, then revert the temporary change and confirm a rerun succeeds.
- [x] 5.4 Set `publish: null` in `packages/bruno-electron/electron-builder-config.js` so electron-builder's CI `onTagOrDraft` default cannot construct a `GitHubPublisher`; verify with `CI=true npm run build:electron:linux` that the stock build succeeds without `GH_TOKEN` and that no `latest-linux.yml` or `app-update.yml` is produced.

## 6. Integration verification

- [x] 6.1 Exercise the whole workflow on a throwaway semver tag (for example `release/v0.0.1`) in a fork, checking job timings, cache hits for `~/.cache/electron` and `~/.cache/electron-builder`, total runtime under the 90-minute build timeout, and release asset names; then delete the test release and tag.
- [x] 6.2 Confirm the repository's Actions workflow-permission setting allows `contents: write` for `GITHUB_TOKEN`; if the publish job receives a 403, record the required settings change in `contributing.md` and re-verify the release job succeeds.

## Workflow follow-up

- Archive this change after the release workflow has been exercised successfully on a real release branch.
- Optional later work, outside this change: signing for the Windows installer, `workflow_dispatch` for retrying a failed run before a release exists, and restoring any test/lint CI that should outlive this cleanup.
