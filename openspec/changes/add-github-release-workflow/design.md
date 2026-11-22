# Design

## Context

See `proposal.md` for motivation and `specs/release-pipeline/spec.md` for the behavioral contract. Constraints found in the repository:

- Seven workflows exist and none of them builds or publishes the desktop app. Existing triggers already use `release/v*` for tests, but those workflows are being deleted.
- `packages/bruno-electron/package.json` is the app metadata electron-builder reads. Its `version` is `2.0.0` at HEAD and at tags `v2.10.0`, `v3.5.2`, `v4.0.0`, and `v4.3.0`, so it cannot name release artifacts correctly on its own.
- `packages/bruno-electron/electron-builder-config.js` targets AppImage, deb, and rpm for x64 + arm64, and nsis for x64 + arm64. `dist:linux` passes `--linux AppImage`, which supplies an explicit target list and overrides that configured list; with no arch flags it builds only the runner arch. `dist:win` passes no target, so the configured nsis x64 + arm64 list applies. `win.sign` is `null`; there is no `publish` block.
- `packages/bruno-electron/package.json` scripts: `dist:linux` = `electron-builder --linux AppImage`, `dist:win` = `electron-builder --win`, plus the bash wrappers `build:electron:linux` / `build:electron:win` that stage the renderer and call those scripts.
- Upstream releases (`v4.3.0`) are live, `target_commitish: release/v4.3.0`, with assets named `bruno_4.3.0_x64_win.exe`, `bruno_4.3.0_x86_64_linux.AppImage`, etc.
- `@lydell/node-pty` resolves its binary at runtime from `@lydell/node-pty-<platform>-<arch>`. `npm ci` on Linux installs both Linux arch packages but not the Windows ones.
- The `ubuntu-24.04` runner image does not preinstall Wine (verified against the published image manifest for 20261116); `ubuntu-latest` is mid-migration to 26.04.
- No package depends on `electron-updater`, so `latest*.yml` has no consumer here.

## Goals / Non-Goals

**Goals:**

- One workflow that turns a `release/v*` push into a live GitHub release with every stock Linux and Windows installer attached.
- Correct artifact names and packaged version without changing the repository's version files.
- Reuse the existing build entry points rather than reimplementing them in YAML.

**Non-Goals:**

- Signing, notarization, snap, flatpak, msi, mac artifacts, auto-update metadata.
- Replacing the deleted test/lint coverage; no test gates are added.
- Publishing from tags or `workflow_dispatch`.

## Decisions

### Four jobs: validate -> linux/windows (parallel) -> publish

`scripts/build-electron.sh` begins with `rm -rf packages/bruno-electron/out`, so running the Linux and Windows builds sequentially in one job would delete the first artifact set. Two parallel build jobs on separate runners avoid that and keep a failure from producing a partial release. A `validate` job fails fast (before ~30 minutes of builds) when the branch name is malformed or the tag/release already exists. `publish` creates the release only after both builds succeed.

Alternative considered: one sequential job moving artifacts to a temp directory between builds. Rejected because it adds custom file shuffling and doubles wall-clock time.

### Trigger and derivation

```yaml
on:
  push:
    branches: ['release/v*']
```

- `TAG = ${GITHUB_REF_NAME#release/}` -> `v4.4.0`
- `VERSION = ${TAG#v}` -> `4.4.0`
- Validation regex: `^v[0-9]+\.[0-9]+\.[0-9]+(-[0-9A-Za-z.-]+)?$` on the derived tag; `validate` exits non-zero if the branch does not match.

Alternatives considered: deriving the version from `package.json` (wrong today), or from a workflow input (incompatible with the push trigger).

### Inject the version by rewriting the app manifest on the runner

Stock `dist:*` scripts forward no arguments, so `-c.extraMetadata.version` cannot be passed through the wrappers. Instead, before building:

```bash
npm pkg set version="$VERSION" --workspace=packages/bruno-electron
```

This changes only the ephemeral checkout, updates `appInfo.version`, and therefore `${version}` in `artifactName`, the packaged `package.json`, and the About dialog. No commit is created.

Alternatives considered: running electron-builder directly with `-c.extraMetadata.version` (verified to deep-merge and preserve `extraMetadata.brunoBuild`, but requires bypassing the wrappers); renaming files after build (leaves the installed app reporting 2.0.0); requiring a version-bump commit (fails against the current repository state).

### Use the stock wrappers, not hand-written staging

Each build job runs the renderer build first and then the stock wrapper for its target, in this order:

```bash
npm run build:web                              # packages/bruno-app/dist
npm run build:electron:linux                   # linux job
npm run build:electron:win                     # windows job
```

`dist:linux` / `dist:win` alone package whatever is already in `packages/bruno-electron/web`; when that directory is empty the packaged app has no renderer. The `build:electron:linux` / `build:electron:win` wrappers perform the existing `bruno-app/dist` -> `bruno-electron/web` copy, path rewrites, and sourcemap removal, then invoke exactly `npm run dist:linux` / `dist:win`. The workflow calls those wrappers, so no staging commands appear in YAML and the stock dist scripts remain the final build step. Only what those scripts emit is published: the x64 AppImage from Linux and the x64, arm64, and combined multi-arch NSIS installers from Windows. The Linux config's deb, rpm, and arm64 targets are not reached.

Alternative considered: replicating the staging steps in the workflow and calling electron-builder with `--linux AppImage --x64` etc. That was the earlier design and is rejected here because it duplicates repository logic and narrows target coverage.

### Parallel build jobs upload Actions artifacts; `publish` downloads them

Each build job uploads only installer files:

| Job | Upload glob | Artifact name |
|---|---|---|
| `build-linux` | `packages/bruno-electron/out/*.AppImage` | `linux-installers` |
| `build-windows` | `packages/bruno-electron/out/*.exe` and `*.exe.blockmap` | `windows-installers` |

The nsis target emits a per-arch installer plus a combined multi-arch installer when both arches are built in one invocation (`bruno_<version>_win.exe`), and a `.blockmap` for each. All of them are uploaded and published; `latest.yml` / `latest-linux.yml` are written to `out/` but excluded by the upload globs.

Unpacked directories and `builder-effective-config.yaml` are not uploaded. `publish` downloads both artifact sets and creates the release with all files, so the release is created once and atomically after both builds pass. The user's "no staging" instruction is about the renderer copy, not about avoiding Actions artifacts; this design does not introduce a separate renderer-staging job.

### Windows build on Ubuntu 24.04 with Wine

- `runs-on: ubuntu-24.04` (pinned; `ubuntu-latest` is rolling to 26.04).
- Install `wine`, `wine64`, and `wine32:i386` after `dpkg --add-architecture i386`. The `wine` metapackage is required because `wine64` / `wine32:i386` alone install the libraries and preloaders without the `/usr/bin/wine` launcher that electron-builder looks for. Electron-builder needs Wine for `rcedit` and for generating/running the NSIS uninstaller. NSIS itself is a native Linux `makensis` from the downloaded toolset, and no signing is configured. The job verifies `command -v wine` and `wine --version` before building so a missing launcher fails fast.
- Force-install Windows PTY binaries because `npm ci` on Linux skips packages with `os: win32`:

```bash
npm i --no-save --force \
  @lydell/node-pty-win32-x64@1.1.0 \
  @lydell/node-pty-win32-arm64@1.1.0
```

- The Linux job verifies that `@lydell/node-pty-linux-x64` is present and fails if it is absent; the only Linux artifact is the x64 AppImage.

### Publish with `gh`, not electron-builder's publisher

There is no `publish` block, and adding one would also generate `latest*.yml` for a nonexistent updater. `publish` instead runs:

```bash
gh release create "$TAG" \
  --target "$GITHUB_REF_NAME" \
  --title "$TAG" \
  --notes "See https://github.com/usebruno/bruno/releases/tag/$TAG" \
  dist/*
```

`--target` keeps `target_commitish` on the release branch, matching upstream. The release is live, with no changelog beyond the upstream link, per the chosen scope. `GITHUB_TOKEN` with `permissions: contents: write` is sufficient; no PAT or third-party action is needed. Tags/releases created with `GITHUB_TOKEN` do not trigger other workflows, which is irrelevant because none remain.

### Duplicate protection

`validate` checks both `gh release view "$TAG"` and `git ls-remote --tags origin "refs/tags/$TAG"`, failing before any build. `publish` re-checks `gh release view` immediately before `gh release create` to close the race; `gh release create` itself fails if the tag/release exists. Existing assets are never modified because there is no `--clobber` path.

### Concurrency, permissions, timeouts

```yaml
permissions:
  contents: read        # validate + both build jobs
# publish job only:
permissions:
  contents: write
concurrency:
  group: release-${{ github.ref_name }}
  cancel-in-progress: false
timeout-minutes: 90     # build jobs; 10 for validate/publish
```

Caches: `actions/setup-node` npm cache in the composite action, plus `~/.cache/electron` and `~/.cache/electron-builder` keyed by `package-lock.json` and the pinned Electron version.

## Risks / Trade-offs

- **No CI remains after deleting seven workflows** -> Accepted by decision. Consequence documented in the proposal: no PR/main unit, e2e, CLI, lint, or benchmark checks, and any branch-protection required checks referencing those jobs will stop reporting.
- **Live release on every release-branch push** -> Duplicate-tag failure makes publication a deliberate act. Failed builds create no tag or release, so retries are safe.
- **Wine regressions on the runner image** -> The Windows job fails loudly rather than silently shipping a Linux-native or unpacked-only artifact. The `wine` metapackage (not just `wine64`/`wine32:i386`) is required for `/usr/bin/wine`; this was hit on the first CI run. Pinning `ubuntu-24.04` avoids the 26.04 transition.
- **Missing PTY binary in a package** -> Explicit presence checks in both build jobs; force-install Windows x64/arm64 and verify Linux x64.
- **`npm pkg set` writes to the checkout** -> Runner-only and never committed; the publish step uses release assets, not the working tree.
- **Runner workflow-permission cap (403 on release creation)** -> Job-level `contents: write` is necessary but can be capped by repository settings; the workflow surfaces the `gh` error, and the fix is a settings change, not a workflow change.
- **Unsigned Windows installer** -> SmartScreen warnings for users; signing is explicitly out of scope.
- **Two builds of the renderer** -> Roughly duplicated `build:web` time in exchange for parallelism; acceptable versus artifact-shuffling complexity.
- **NSIS blockmaps and update files** -> `.blockmap` files are uploaded with the installers and are inert without an updater. `latest.yml` / `latest-linux.yml` are produced by electron-builder but excluded from the upload globs.

## Migration Plan

1. Delete the seven workflow files in one commit and add `.github/workflows/release.yml`.
2. Test on a throwaway branch/tag (for example `release/v0.0.1`) in a fork: confirm validation, both jobs, asset names, and the release body.
3. Delete the throwaway release/tag before using a real version.
4. Rollback is `git revert` of the workflow commit, which restores the previous workflow files. Deleting a published release/tag is a manual `gh release delete` + `git push --delete`.

## Open Questions

None blocking. Optional later additions that do not change the spec: `workflow_dispatch` for rebuilding a failed run before a release exists, and signing once a certificate is available.
