# Proposal

## Why

This repository has no release automation: the seven existing workflows only run tests, lint, benchmarks, and a manual CLI check, so release binaries must be produced and published by hand. A push to a `release/*` branch should deterministically build the desktop artifacts and publish them to GitHub, using the build commands the repository already ships.

## What Changes

- **BREAKING**: delete all seven existing workflow files (`tests-linux.yml`, `tests-macos.yml`, `tests-windows.yml`, `lint-checks.yml`, `benchmarks.yml`, `flaky-test-detector.yml`, `npm-bru-cli.yml`). No CI remains on pull requests or `main`.
- Add one workflow, `.github/workflows/release.yml`, triggered by pushes to branches matching `release/v*`.
- Derive the release tag from the branch name (`release/v4.4.0` -> `v4.4.0`) and the version from the tag (`4.4.0`), and inject that version into `packages/bruno-electron/package.json` before building so artifact names and packaged metadata match the release.
- Build the renderer with `npm run build:web`, then use the repository's stock entry points `npm run build:electron:linux` and `npm run build:electron:win`, which invoke the stock `dist:linux` / `dist:win` scripts and keep their configured targets and architectures.
- Publish every artifact those builds produce (Linux AppImage, deb, and rpm for x64 and arm64; Windows NSIS installers for x64 and arm64) to a new live GitHub release with an empty body linking back to the upstream release notes.
- If the tag or release already exists, fail the run instead of replacing published binaries.
- Leave `.github/actions/**` and `.github/scripts/**` in place; they become unused but are outside this change.

## Capabilities

### New Capabilities

- `release-pipeline`: how a push to a release branch becomes built Linux and Windows artifacts attached to a GitHub release, including version derivation, artifact coverage, and duplicate-release handling.

### Modified Capabilities

None.

## Impact

- `.github/workflows/`: seven files deleted, one file added.
- `packages/bruno-electron/package.json`: version is rewritten on the runner before building (not committed).
- Release assets: `*.AppImage`, `*.deb`, `*.rpm` (x64 + arm64) and `*.exe` (x64 + arm64), plus any blockmap files the NSIS target emits.
- Runners: Ubuntu with Wine for the Windows target and `@lydell/node-pty-win32-*` binaries installed into the dependency tree.
- Loss of all existing CI: no unit, e2e, CLI, lint, or benchmark checks on pull requests or `main`; branch-protection required checks tied to those workflows will stop reporting.
- Artifacts remain unsigned (`win.sign: null`), and no auto-update metadata (`latest*.yml`) is published because this codebase has no `electron-updater`.
