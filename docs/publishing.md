# Publishing a plugin

This guide explains how plugin repositories publish npm packages under the `@gcds-extensions` scope using shared GitHub Actions workflows. Two release tools are supported:

- **Changesets**: best when you want explicit versioning and release notes via checked-in changeset files.
- **Release Please**: best when you want conventional-commit driven release PRs and automatic tagging/releases.

## Shared workflows

Plugin repositories should call one of these reusable workflows instead of copying workflow logic.

| Tool | Reusable workflow |
| --- | --- |
| Changesets | [`./.github/workflows/publish.yml`](../.github/workflows/publish.yml) |
| Release Please | [`./.github/workflows/publish-release-please.yml`](../.github/workflows/publish-release-please.yml) |

## Plugin repository workflow

In each plugin repository, create a release workflow that calls one of the shared workflows:

<!-- tab: Changesets -->

Create `.github/workflows/publish.yml`:

```yaml
name: Release

on:
  push:
    branches:
      - main

permissions:
  contents: write
  pull-requests: write
  id-token: write

concurrency:
  group: release-${{ github.repository }}
  cancel-in-progress: false

jobs:
  release:
    uses: gcds-extensions/plugins/.github/workflows/publish.yml@v1
    permissions:
      contents: write
      pull-requests: write
      id-token: write
    with:
      node-version: "24"
```

<!-- tab: Release Please -->

Create `.github/workflows/publish.yml` (Release Please variant):

```yaml
name: Release

on:
  push:
    branches:
      - main

permissions:
  contents: write
  pull-requests: write
  id-token: write

concurrency:
  group: release-${{ github.repository }}
  cancel-in-progress: false

jobs:
  release:
    uses: gcds-extensions/plugins/.github/workflows/publish-release-please.yml@v1
    permissions:
      contents: write
      pull-requests: write
      id-token: write
    with:
      node-version: "24"
      release-type: node
```

<!-- end tabs -->

## Where actual publishing is defined

The shared workflow runs your publish script, so each plugin defines release behavior in its own `package.json` scripts.

<!-- tab: Changesets -->

The Changesets workflow runs:

```yaml
with:
  publish: npm run release
```

Example `package.json`:

```json
{
  "name": "@gcds-extensions/code-display",
  "version": "1.0.0",
  "scripts": {
    "build": "your-build-command",
    "test": "your-test-command",
    "release": "changeset publish",
  }
}
```

Use `release` for stable by default.

### Publishing overrides (alpha/beta)
For plugins that need staged releases, override the `publish` input:

```yaml
jobs:
  release:
    uses: gcds-extensions/plugins/.github/workflows/publish.yml@v1
    permissions:
      contents: write
      pull-requests: write
      id-token: write
    with:
      node-version: "24"
      publish: npm run release:alpha  # or release:beta
```

Example `package.json` with overrides:
```json
{
  "name": "@gcds-extensions/code-display",
  "version": "0.1.0-alpha",
  "scripts": {
    "build": "your-build-command",
    "test": "your-test-command",
    "release": "changeset publish",
    "release:alpha": "changeset publish --tag alpha",
    "release:beta": "changeset publish --tag beta"
  }
}
```

<!-- tab: Release Please -->

The Release Please workflow runs:

```yaml
with:
  publish: npm run release
```

Example `package.json`:

```json
{
  "name": "@gcds-extensions/code-display",
  "version": "1.0.0",
  "scripts": {
    "build": "your-build-command",
    "test": "your-test-command",
    "release": "npm publish --provenance --access public"
  }
}
```

Use `release` for stable by default.

### Publishing overrides (alpha/beta)
Release Please can still run alternate publish scripts via `publish`, but it does not manage prerelease version progression/tags in the same way as Changesets. Prefer prerelease flows with Changesets when you need structured alpha/beta versioning.

```yaml
jobs:
  release:
    uses: gcds-extensions/plugins/.github/workflows/publish-release-please.yml@v1
    permissions:
      contents: write
      pull-requests: write
      id-token: write
    with:
      node-version: "24"
      release-type: node
      publish: npm run release:alpha  # or release:beta
```

Example `package.json` with overrides:

```json
{
  "name": "@gcds-extensions/code-display",
  "version": "1.0.0",
  "scripts": {
    "build": "your-build-command",
    "test": "your-test-command",
    "release": "npm publish --provenance --access public",
    "release:alpha": "npm publish --provenance --access public --tag alpha",
    "release:beta": "npm publish --provenance --access public --tag beta"
  }
}
```

<!-- end tabs -->

## npm access and ownership model

Use least-privilege access:

1. Invite maintainers to the npm organization.
2. Add maintainers to a dedicated `publishers` team (separate from `developers`).
3. Grant package publish rights to `publishers` only for required scoped packages.

Do not share personal npm credentials or reusable publish tokens.

## Bootstrap first publish (`0.0.0-alpha`)

A GCDS team member performs a one-time bootstrap publish so the package exists in npm:

1. Set package version to `0.0.0-alpha`.
2. Run `npm ci`.
3. Run `npm run build`.
4. Run `npm publish --access public --tag alpha`.

After this bootstrap publish, switch the package to Trusted Publishing.

## Trusted Publishing setup (npm)

Trusted Publishing is configured in npm package settings (not only in YAML):

1. Open npm package settings for `@gcds-extensions/<plugin>`.
2. Add a Trusted Publisher for GitHub repository `gcds-extensions/<plugin>`.
3. Select the plugin release workflow file (for example `.github/workflows/release.yml`).
4. Save settings.

Once configured, GitHub Actions uses OIDC (`id-token: write`) for publish access and `NPM_TOKEN` is not required.

> Note: In the Release Please shared workflow, the publish job runs from the release tag (`tag_name`) and publishes the tagged commit.
