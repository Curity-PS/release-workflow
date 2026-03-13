# ps-release-workflow

A reusable GitHub Actions workflow for testing and releasing Gradle-based Curity plugins. It runs unit and integration tests, generates a changelog using [Conventional Commits](https://www.conventionalcommits.org/), and publishes a GitHub Release with the built plugin artifact. Releases are version controlled using git tags.

## Usage

Create a workflow file in your repository (e.g. `.github/workflows/release.yml`) that calls this reusable workflow. Use `workflow_dispatch` inputs so you can choose what to run from the GitHub UI:

```yaml
name: Test & Release

on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      run_unit_tests:
        description: 'Run unit tests'
        type: boolean
        default: true
      run_integration_tests:
        description: 'Run integration tests'
        type: boolean
        default: true
      make_release:
        description: 'Create a release'
        type: boolean
        default: false

jobs:
  release:
    uses: curity-ps/release-workflow/.github/workflows/release.yml@main
    with:
      skip_test: ${{ github.event_name == 'workflow_dispatch' && !inputs.run_unit_tests }}
      skip_integration_test: ${{ github.event_name == 'workflow_dispatch' && !inputs.run_integration_tests }}
      make_release: ${{ github.event_name != 'workflow_dispatch' || inputs.make_release }}
    secrets:
      PACKAGE_TOKEN: ${{ secrets.PACKAGE_TOKEN }}
      LICENSE_KEY: ${{ secrets.LICENSE_KEY }}
```

When triggered by a push to `main`, all tests run and a release is created (the defaults). When triggered manually via `workflow_dispatch`, you pick what to do using the checkboxes in the GitHub UI.

## Inputs

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `skip_test` | `boolean` | `false` | Skip unit tests |
| `skip_integration_test` | `boolean` | `false` | Skip integration tests |
| `version_from_gradle` | `boolean` | `false` | Use the Gradle project version (from `build.gradle`, `gradle.properties`, etc.) instead of the conventional-changelog version |
| `make_release` | `boolean` | `true` | Create a GitHub release and tag |

### Example: skip integration tests

```yaml
jobs:
  release:
    uses: curity-ps/release-workflow/.github/workflows/release.yml@main
    with:
      skip_integration_test: true
    secrets:
      PACKAGE_TOKEN: ${{ secrets.PACKAGE_TOKEN }}
```

### Example: skip all tests

```yaml
jobs:
  release:
    uses: curity-ps/release-workflow/.github/workflows/release.yml@main
    with:
      skip_test: true
      skip_integration_test: true
```

### Example: use the Gradle project version

```yaml
jobs:
  release:
    uses: curity-ps/release-workflow/.github/workflows/release.yml@main
    with:
      version_from_gradle: true
    secrets:
      PACKAGE_TOKEN: ${{ secrets.PACKAGE_TOKEN }}
```

## Secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `PACKAGE_TOKEN` | No | GitHub token used for Gradle dependency resolution and creating releases. Falls back to the automatic `secrets.GITHUB_TOKEN` if not provided. Use a PAT if you need access to packages in other repositories. |
| `LICENSE_KEY` | **Yes** (when integration tests are enabled) | License key passed to integration tests. The workflow will fail if this secret is missing and `skip_integration_test` is `false`. |

## Requirements

The calling repository must be a Gradle project using the [curity-plugin-dev](https://github.com/Curity-PS/curity-plugin-dev) gradle plugin, or with the following tasks available:

- `test` — runs unit tests
- `integrationTest` — runs integration tests
- `createRelease` — builds the plugin and produces `build/distributions/plugin-artifacts.zip`

The project must use [Conventional Commits](https://www.conventionalcommits.org/) for commit messages. The workflow uses these to determine the next version number and generate the changelog. If no releasable commits are found since the last tag, the release step is skipped.

## What it does

1. **build_and_package** — Checks out the code, sets up JDK 21 and Gradle, then runs unit and integration tests (unless skipped).
2. **make_release** — Determines the next semantic version from conventional commits, runs `./gradlew createRelease`, and creates a GitHub Release with the generated changelog and artifact zip.
