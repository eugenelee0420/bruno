# Spec Delta

## Purpose

Turn a push to a release branch into a complete, correctly versioned set of Linux and Windows desktop artifacts published to the GitHub release for that tag.

## ADDED Requirements

### Requirement: Release trigger and tag derivation

The release workflow SHALL run only for pushes to branches matching `release/v<major>.<minor>.<patch>` and SHALL derive the tag by removing the `release/` prefix. A branch that matches the `release/` trigger but not the version format SHALL fail before any build starts.

#### Scenario: Valid release branch push

- **WHEN** a commit is pushed to `release/v4.4.0`
- **THEN** the run targets tag `v4.4.0` and version `4.4.0`

#### Scenario: Non-release branch push

- **WHEN** a commit is pushed to `main` or `feature/x`
- **THEN** the release workflow does not run

#### Scenario: Malformed release branch

- **WHEN** a commit is pushed to `release/test`
- **THEN** the run fails before any build starts

### Requirement: Artifacts carry the release version

Packaged application metadata and artifact file names SHALL use the version derived from the tag without its leading `v`.

#### Scenario: Version injected into build

- **WHEN** tag `v4.4.0` is built
- **THEN** artifacts are named `bruno_4.4.0_*` and the packaged application reports version `4.4.0`

### Requirement: Artifact coverage

Each successful run SHALL produce, through the repository's stock `dist:linux` and `dist:win` commands, AppImage, deb, and rpm packages for Linux x64 and arm64, and NSIS installers for Windows x64 and arm64.

#### Scenario: Linux artifacts

- **WHEN** the Linux build succeeds
- **THEN** it produces `x86_64` and `arm64` AppImages, `amd64` and `arm64` deb packages, and `x86_64` and `aarch64` rpm packages

#### Scenario: Windows artifacts

- **WHEN** the Windows build succeeds
- **THEN** it produces NSIS installers for x64 and arm64

### Requirement: Complete release publication

A successful run SHALL create one live GitHub release tagged with the derived tag, targeted at the pushed release branch, and SHALL attach every installer the builds produced. The release body SHALL link to the upstream release notes for the same tag.

#### Scenario: Release created with all artifacts

- **WHEN** both the Linux and Windows builds succeed for `v4.4.0`
- **THEN** a live release `v4.4.0` exists with the Linux and Windows installers attached and a body linking to `https://github.com/usebruno/bruno/releases/tag/v4.4.0`

#### Scenario: Partial build failure

- **WHEN** either build fails
- **THEN** no release and no tag are created

### Requirement: Duplicate release rejection

If the derived tag or its release already exists, the run SHALL fail before publishing and SHALL NOT modify existing assets.

#### Scenario: Re-push to a released tag

- **WHEN** `release/v4.4.0` is pushed again after release `v4.4.0` exists
- **THEN** the run fails and the existing release assets are unchanged

#### Scenario: Retry after a failed build

- **WHEN** a previous run failed before creating the release and `release/v4.4.0` is pushed again
- **THEN** the run may proceed and create the release

### Requirement: Architecture-correct native binaries

Every published package SHALL contain the PTY binary for its own platform and architecture and SHALL NOT depend on the runner host's platform binary.

#### Scenario: Windows arm64 package

- **WHEN** the Windows arm64 installer is packaged
- **THEN** its dependency tree contains the `win32-arm64` PTY binary

#### Scenario: Linux arm64 package

- **WHEN** the Linux arm64 AppImage is packaged
- **THEN** its dependency tree contains the `linux-arm64` PTY binary
