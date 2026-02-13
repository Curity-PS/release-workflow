# ps-release-workflow

A reusable GitHub Actions workflow for testing and releasing Gradle-based Curity plugins. It runs unit and integration tests, generates a changelog using [Conventional Commits](https://www.conventionalcommits.org/), and publishes a GitHub Release with the built plugin artifact. Releases are version controlled using git tags.

## Usage

Create a workflow file in your repository (e.g. `.github/workflows/release.yml`) that calls this reusable workflow:

```yaml
name: Release

on:
  push:
    branches: [main]

jobs:
  release:
    uses: curity-ps/release-workflow/.github/workflows/release.yml@main
    secrets:
      PACKAGE_TOKEN: ${{ secrets.PACKAGE_TOKEN }}
      LICENSE_KEY: ${{ secrets.LICENSE_KEY }}
```

## Inputs

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `skip_test` | `boolean` | `false` | Skip unit tests |
| `skip_integration_test` | `boolean` | `false` | Skip integration tests |

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

## Secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `PACKAGE_TOKEN` | No | GitHub token used for Gradle dependency resolution and creating releases. Falls back to the automatic `secrets.GITHUB_TOKEN` if not provided. Use a PAT if you need access to packages in other repositories. |
| `LICENSE_KEY` | No | License key passed to integration tests. Only required when integration tests are enabled. |

## Requirements

The calling repository must be a Gradle project using the [curity-plugin-dev](https://github.com/Curity-PS/curity-plugin-dev) gradle plugin, or with the following tasks available:

- `test` — runs unit tests
- `integrationTest` — runs integration tests
- `createRelease` — builds the plugin and produces `build/distributions/plugin-artifacts.zip`

The project must use [Conventional Commits](https://www.conventionalcommits.org/) for commit messages. The workflow uses these to determine the next version number and generate the changelog. If no releasable commits are found since the last tag, the release step is skipped.

## What it does

1. **build_and_package** — Checks out the code, sets up JDK 21 and Gradle, then runs unit and integration tests (unless skipped).
2. **make_release** — Determines the next semantic version from conventional commits, runs `./gradlew createRelease`, and creates a GitHub Release with the generated changelog and artifact zip.
