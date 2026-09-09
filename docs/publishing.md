# Publishing a plugin

Every `@gcds-extensions/*` plugin releases through two shared workflows hosted
in this repository. Your plugin repo holds two small caller files; the logic
lives here, so a fix here reaches every plugin.

- [What to expect](#what-to-expect)
- [What you will need](#what-you-will-need)
- [Setup](#setup)
- [How to use it day to day](#how-to-use-it-day-to-day)
- [How publishing happens](#how-publishing-happens)
- [Workflow reference](#workflow-reference)
- [Troubleshooting](#troubleshooting)

## What to expect

```
commit to main ──> release-generator ──> opens/updates a release PR
                                              │
                                          you merge it
                                              │
                                              ▼
                              version bumped, CHANGELOG written,
                                  tag vX.Y.Z + GitHub release
                                              │
                                              ▼
                                    publish ──> npm
```

Three things follow from this shape:

- **You never edit the version by hand.** release-please derives it from your
  conventional commit messages and writes it into `package.json` and
  `CHANGELOG.md` inside the release PR.
- **Merging the release PR is the release.** That merge is what creates the
  tag, and the tag is what triggers publishing. Nothing else publishes.
- **Tests do not run in the publish path.** They run on pull requests. By the
  time a tag exists, the code has already been reviewed and tested.

## What you will need

| | Requirement | Notes |
| --- | --- | --- |
| Repo | Lives in the `gcds-extensions` GitHub org | The shared workflows are public, so no extra org settings are needed |
| Repo | `package.json` with `name`, `version`, `files`, and `repository.url` | `repository.url` must match the GitHub repo or provenance fails |
| Repo | A committed `package-lock.json` | The workflow uses `npm ci` |
| Repo | Know where your `package.json` lives | Root needs no extra config; a package in a subdirectory needs `working-directory` and/or `package-path` — see [Publishing from a subdirectory](#publishing-from-a-subdirectory) |
| Repo | A `build` script | The workflow runs `npm run build` before publishing |
| Repo | [Conventional commits](https://www.conventionalcommits.org/) on `main` | `feat:` bumps the minor, `fix:` the patch. Non-conventional commits are ignored for versioning |
| Secrets | `GCDS_RELEASE_BOT_APP_ID`, `GCDS_RELEASE_BOT_PRIVATE_KEY` | Ask the GCDS team. Required, not optional — see below |
| npm | The package already exists on npm | Trusted Publishing is configured per package, so a one-time bootstrap publish comes first |
| npm | A Trusted Publisher pointed at your repo and caller filename | This is the step that most often goes wrong |

### Why the App credentials are required

Tags and pull requests created with the default `GITHUB_TOKEN` **do not trigger
other workflows**. Without the release bot App: your CI would not run on the
release PR, and the tag created when you merge it would never fire the publish
workflow. Publishing would silently never happen. There is no fallback.

## Setup

### 1. Add the two caller workflows

Create `.github/workflows/release-generator.yml`:

```yaml
name: Release Generator

on:
  workflow_dispatch:
  push:
    branches: [main]

permissions:
  contents: write
  pull-requests: write

concurrency:
  group: release-generator-${{ github.ref }}
  cancel-in-progress: false

jobs:
  release:
    uses: gcds-extensions/plugins/.github/workflows/release-generator.yml@<sha>
    permissions:
      contents: write
      pull-requests: write
    secrets:
      release_app_id: ${{ secrets.GCDS_RELEASE_BOT_APP_ID }}
      release_app_private_key: ${{ secrets.GCDS_RELEASE_BOT_PRIVATE_KEY }}
    with:
      release-type: node
```

Create `.github/workflows/compile-and-publish.yml`:

```yaml
name: Publish packages

on:
  workflow_dispatch:
    inputs:
      ref:
        description: Existing tag to publish, e.g. v1.2.0
        required: true
        type: string
  push:
    tags: ["v*"]

permissions:
  contents: read
  id-token: write

concurrency:
  group: publish-${{ inputs.ref || github.ref_name }}
  cancel-in-progress: false

jobs:
  publish:
    uses: gcds-extensions/plugins/.github/workflows/publish.yml@<sha>
    permissions:
      contents: read
      id-token: write
    with:
      ref: ${{ inputs.ref || github.ref_name }}
      node-version: "24"
```

Keep the publish caller's filename stable once you register it with npm.

Those two blocks are all a single-package repository needs. If your
`package.json` is **not** at the repository root, add one or two more inputs
first — see [Publishing from a subdirectory](#publishing-from-a-subdirectory).

### 2. Pin the shared workflows by SHA

Replace `<sha>` in both files with a full commit SHA:

```bash
gh api repos/gcds-extensions/plugins/commits/main --jq '.sha'
```

Use a full 40-character SHA with the version in a trailing comment:

```yaml
uses: gcds-extensions/plugins/.github/workflows/publish.yml@e2fbc1a…8c2 # v1
```

A SHA cannot be moved; branch and tag refs can be retargeted by a force-push,
and these workflows hold publishing rights. Add the `github-actions` ecosystem
to `.github/dependabot.yml` and Dependabot will keep the pin current.

### 3. Add the repository secrets

Add `GCDS_RELEASE_BOT_APP_ID` and `GCDS_RELEASE_BOT_PRIVATE_KEY` to the
repository, and have the GCDS team install the release bot App on it with
`contents: write` and `pull requests: write`.

### 4. Bootstrap the package on npm

Trusted Publishing is configured per package, so the package must exist first.
A GCDS team member does this once, locally:

```bash
npm version 0.0.0-alpha --no-git-tag-version
npm ci
npm run build
npm publish --access public --tag alpha
```

Then set the version in `package.json` back to a clean `0.0.0` and commit it.
release-please parses that field; a lingering `-alpha` produces confusing
version proposals.

### 5. Configure Trusted Publishing

In npm package settings for `@gcds-extensions/<plugin>`:

1. Add a Trusted Publisher for GitHub repository `gcds-extensions/<plugin>`.
2. Workflow filename: **`compile-and-publish.yml`** — filename only, no path.
3. Leave environment blank.

> npm validates the **caller** workflow in your plugin repo, not the shared
> workflow that runs `npm publish`. Register your own file, never `publish.yml`
> from this repository. After this, no npm token is needed anywhere.

### 6. Anchor the changelog

If the repo has no tags yet, release-please treats the entire history as
unreleased and writes a very long first changelog. Create an empty anchor:

```bash
git tag v0.0.0 && git push origin v0.0.0
```

Then create a GitHub release for that tag. Do this before the first run.

## How to use it day to day

Write conventional commits on `main`:

| Commit | Effect below 1.0.0 | Effect at 1.0.0+ |
| --- | --- | --- |
| `fix: …` | patch (0.1.0 → 0.1.1) | patch |
| `feat: …` | minor (0.1.0 → 0.2.0) | minor |
| `feat!: …` or `BREAKING CHANGE:` in the body | minor | major |
| `chore: …`, `docs: …`, `ci: …` | no release | no release |

Anything not matching the convention is ignored for versioning and left out of
the changelog. To force a specific version, put `Release-As: 1.0.0` in a commit
body.

release-please keeps **one** open release PR and rewrites it as you land more
commits. Merge it when you want to cut a release; leave it open otherwise.

## How publishing happens

1. You merge the release PR.
2. That merge is a push to `main`, so `release-generator` runs again, sees the
   merged PR, and creates the tag and GitHub release.
3. The new tag matches `v*`, which triggers `compile-and-publish`.
4. `check-version` reads the version from `package.json` at that tag and asks
   npm whether it is already published. If it is, publishing is skipped and the
   run ends green — re-runs are safe.
5. `publish` checks out the tag, runs `npm ci` and `npm run build`, then
   `npm publish --ignore-scripts --access public`. Authentication is OIDC;
   npm attaches a provenance attestation automatically.

To publish an existing tag manually — for example after fixing a Trusted
Publisher misconfiguration — run the **Publish packages** workflow from the
Actions tab and give it the tag.

## Workflow reference

### `release-generator.yml`

| Input | Default | Purpose |
| --- | --- | --- |
| `release-type` | `node` | release-please strategy |
| `path` | `.` | Release from a subdirectory (monorepos) |

| Secret | Required | Purpose |
| --- | --- | --- |
| `release_app_id` | yes | Release bot App ID |
| `release_app_private_key` | yes | Release bot private key |

Outputs: `release_created`, `tag_name`.

### `publish.yml`

| Input | Default | Purpose |
| --- | --- | --- |
| `ref` | — (required) | Tag to check out and publish |
| `node-version` | `24` | Must be >= 22.14.0 for Trusted Publishing |
| `working-directory` | `.` | Where `npm ci` and `npm run build` run |
| `package-path` | `.` | Directory containing the `package.json` to publish |
| `npm-tag` | `latest` | npm dist-tag, e.g. `alpha`, `beta`, `next` |

No secrets — publishing is OIDC only.

#### Publishing from a subdirectory

`working-directory` and `package-path` are separate because the two monorepo
layouts need different answers, and the wrong one fails confusingly.

**Workspaces monorepo** — one lockfile at the root, packages under
`packages/*`. Install and build must happen at the root, because that is the
only place `package-lock.json` exists. Publish from the package:

```yaml
with:
  ref: ${{ inputs.ref || github.ref_name }}
  package-path: packages/web        # working-directory stays "."
```

**Standalone package in a subdirectory** — its own `package-lock.json`, not a
workspace. Everything happens in that directory:

```yaml
with:
  ref: ${{ inputs.ref || github.ref_name }}
  working-directory: plugins/code-display
  package-path: plugins/code-display
```

Setting only `package-path` in the second case makes `npm ci` fail at the root
with a missing lockfile. Single-package repositories need neither input.

## Troubleshooting

**No release PR appears.** Check that commits since the last release use
conventional prefixes; `chore:` and unprefixed commits produce no release. Check
the App secrets exist and the App is installed on the repo.

**The release PR merged but nothing published.** The tag must be created by the
App token. If the run used `GITHUB_TOKEN`, the tag push does not trigger
workflows. Confirm the tag exists, then publish it manually via
`workflow_dispatch`.

**`ENEEDAUTH` or 401 on publish.** Usually the wrong workflow filename
registered with npm (it must be the caller in your repo), a caller missing
`id-token: write`, or npm older than 11.5.1.

**`npm error 403 … cannot publish over previously published version`.** The
`check-version` guard should prevent this; if you see it, the tag's
`package.json` version differs from what was released.

**`npm ci` fails with "can only install with an existing package-lock.json".**
The install is running somewhere without a lockfile. For a package in a
subdirectory, set `working-directory` to the directory holding its
`package-lock.json` — setting only `package-path` leaves the install at the
repository root. See [Publishing from a subdirectory](#publishing-from-a-subdirectory).

**Provenance fails.** `repository.url` in `package.json` must match the GitHub
repository.

**The first changelog is enormous.** No anchor tag existed. See step 6.
