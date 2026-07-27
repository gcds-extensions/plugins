# Publishing a plugin

This guide explains how plugin repositories publish npm packages under the `@gcds-extensions` scope using the shared GitHub Actions workflow.

## Shared workflow

This repository provides the reusable workflow at:

`./.github/workflows/publish.yml`

Plugin repositories should call it instead of copying workflow logic.

## Plugin repository workflow

In each plugin repository, create `.github/workflows/release.yml`:

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

## Where actual publishing is defined

The shared workflow runs:

```yaml
with:
  publish: npm run release
```

That means each plugin defines its release behavior in its own `package.json` scripts.

Example `package.json`:

```json
{
  "name": "@gcds-extensions/code-display",
  "version": "0.0.0-alpha",
  "scripts": {
    "build": "your-build-command",
    "test": "your-test-command",
    "release": "changeset publish",
    "release:alpha": "changeset publish --tag alpha",
    "release:beta": "changeset publish --tag beta"
  }
}
```

Use `release` for stable by default. Add `release:alpha` and `release:beta` scripts when staged publishing is needed.

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
