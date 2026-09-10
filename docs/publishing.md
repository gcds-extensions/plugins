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
- **Tests do not run in the publish path.** They belong on pull requests, so a
  failure blocks the merge rather than blocking a release after the fact. You
  supply that workflow — see [step 7](#7-run-your-tests-on-pull-requests).

## What you will need

| | Requirement | Notes |
| --- | --- | --- |
| Repo | Lives in the `gcds-extensions` GitHub org | The shared workflows are public, so no extra org settings are needed |
| Repo | `package.json` with `name`, `version`, `files`, and `repository.url` | `repository.url` must match the GitHub repo or provenance fails |
| Repo | A committed `package-lock.json` | The workflow uses `npm ci` |
| Repo | Know where your `package.json` lives | Root needs no extra config; a package in a subdirectory needs `working-directory` and/or `package-path` — see [Publishing from a subdirectory](#publishing-from-a-subdirectory) |
| Repo | A `build` script | The workflow runs `npm run build` before publishing |
| Repo | A workflow that runs tests on pull requests | The shared publish workflow does not run tests — see [step 7](#7-run-your-tests-on-pull-requests) |
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

Work through these in order. Step 1 must come before step 2, or pushing the
anchor tag will fire the publish workflow you just installed.

### 1. Anchor the repository history

If the repo has no tags yet, release-please treats the entire history as
unreleased and writes a very long first changelog. Create an anchor at the
current version:

```bash
git tag v0.0.0 && git push origin v0.0.0
```

Then create a GitHub release for that tag.

> Do this **before** adding the workflows in step 2. The anchor tag matches the
> publish workflow's `v*` trigger, so pushing it afterwards starts a publish run
> for a version you did not intend to release.

### 2. Add the two caller workflows

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

**Keep `node-version` at `24` or higher.** Trusted Publishing needs npm
>= 11.5.1, and the shared workflow uses whatever npm ships with the Node you
ask for. Node 22 still ships npm 10.x, so setting `node-version: "22"` fails at
the publish step even though Node 22 clears npm's documented Node floor.

### 3. Pin the shared workflows by SHA

Replace `<sha>` in both files with a full 40-character commit SHA, keeping the
version in a trailing comment:

```bash
gh api repos/gcds-extensions/plugins/commits/main --jq '.sha'
```

```yaml
uses: gcds-extensions/plugins/.github/workflows/publish.yml@0000000000000000000000000000000000000000 # v1
```

A SHA cannot be moved; branch and tag refs can be retargeted by a force-push,
and these workflows hold publishing rights. Add the `github-actions` ecosystem
to `.github/dependabot.yml` and Dependabot will keep the pin current.

### 4. Add the repository secrets

Add `GCDS_RELEASE_BOT_APP_ID` and `GCDS_RELEASE_BOT_PRIVATE_KEY` to the
repository, and have the GCDS team install the release bot App on it with
`contents: write` and `pull requests: write`.

### 5. Bootstrap the package on npm

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

Check what is already published before your first real release — the version in
`package.json` must be higher than anything on npm, or `check-version` will skip
the release as already published:

```bash
npm view @gcds-extensions/<plugin> versions
```

### 6. Configure Trusted Publishing

In npm package settings for `@gcds-extensions/<plugin>`:

1. Add a Trusted Publisher for GitHub repository `gcds-extensions/<plugin>`.
2. Workflow filename: **`compile-and-publish.yml`** — filename only, no path.
3. Leave environment blank.

> npm validates the **caller** workflow in your plugin repo, not the shared
> workflow that runs `npm publish`. Register your own file, never `publish.yml`
> from this repository. After this, no npm token is needed anywhere.

### 7. Run your tests on pull requests

The shared publish workflow deliberately does not run tests — by the time a tag
exists the code has already been merged, so a failing test there would block a
release without preventing the bad merge. Add your own workflow so tests gate
the pull request instead:

```yaml
name: Run Tests

on:
  workflow_dispatch:
  pull_request:
    types: [opened, reopened, synchronize]

permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@<sha> # v7
        with:
          persist-credentials: false
      - uses: actions/setup-node@<sha> # v7
        with:
          node-version: "24"
          cache: npm
      - run: npm ci
      - run: npm run build
      - run: npm test
```

This is also what the release bot App buys you: a release PR opened with the
default `GITHUB_TOKEN` would not trigger this workflow at all.

## How to use it day to day

Write conventional commits on `main`:

| Commit | Effect below 1.0.0 | Effect at 1.0.0+ |
| --- | --- | --- |
| `fix: …` | patch (0.1.0 → 0.1.1) | patch |
| `feat: …` | minor (0.1.0 → 0.2.0) | minor |
| `feat!: …` or `BREAKING CHANGE:` in the body | **major (0.0.1 → 1.0.0)** | major |
| `chore: …`, `docs: …`, `ci: …` | no release | no release |

Anything not matching the convention is ignored for versioning and left out of
the changelog. To force a specific version, put `Release-As: 1.0.0` in a commit
body.

> **Below 1.0.0, one breaking change ships 1.0.0.** release-please's
> `bump-minor-pre-major` defaults to `false`, so a `feat!:` at `0.0.1` proposes
> `1.0.0`, not `0.1.0`. If you are not ready to declare the API stable, switch
> to manifest mode and set `bump-minor-pre-major: true` — see
> [Customizing releases](#customizing-releases).

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
   `npm publish --ignore-scripts --access public --tag <npm-tag>`.
   Authentication is OIDC; npm attaches a provenance attestation automatically.

To publish an existing tag manually — for example after fixing a Trusted
Publisher misconfiguration — run the **Publish packages** workflow from the
Actions tab and give it the tag.

## Workflow reference

### `release-generator.yml`

| Input | Default | Purpose |
| --- | --- | --- |
| `release-type` | `node` | release-please strategy. `""` switches to manifest mode |
| `config-file` | `release-please-config.json` | Manifest mode only |
| `manifest-file` | `.release-please-manifest.json` | Manifest mode only |
| `path` | `.` | Release from a subdirectory (monorepos). Simple mode only |

| Secret | Required | Purpose |
| --- | --- | --- |
| `release_app_id` | yes | Release bot App ID |
| `release_app_private_key` | yes | Release bot private key |

Outputs: `release_created`, `tag_name`.

#### Customizing releases

Simple mode (`release-type: node`) needs no config files and covers most
plugins. It cannot reach release-please's tuning options, because those live in
a config file rather than as action inputs. Switch to manifest mode when you
need any of:

| Option | Default | Why you might change it |
| --- | --- | --- |
| `bump-minor-pre-major` | `false` | `true` keeps breaking changes on the minor while below 1.0.0, instead of jumping to 1.0.0 |
| `bump-patch-for-minor-pre-major` | `false` | `true` makes `feat:` bump the patch while below 1.0.0 |
| `changelog-sections` | feat/fix/breaking shown | Show `perf:`, `refactor:` or `docs:` in the changelog, or hide types you do not want |
| `extra-files` | none | Bump the version string in files beyond `package.json` |
| `include-component-in-tag` | `true` | **Must be `false`.** See the warning below |

To switch, pass an empty `release-type` and commit two files:

```yaml
    with:
      release-type: ""
```

`release-please-config.json`:

```json
{
  "$schema": "https://raw.githubusercontent.com/googleapis/release-please/main/schemas/config.json",
  "bump-minor-pre-major": true,
  "include-component-in-tag": false,
  "changelog-sections": [
    { "type": "feat", "section": "Features" },
    { "type": "fix", "section": "Bug Fixes" },
    { "type": "perf", "section": "Performance" },
    { "type": "refactor", "section": "Refactors", "hidden": true },
    { "type": "docs", "section": "Documentation", "hidden": true },
    { "type": "test", "section": "Tests", "hidden": true },
    { "type": "build", "section": "Build", "hidden": true },
    { "type": "ci", "section": "CI", "hidden": true },
    { "type": "chore", "section": "Miscellaneous", "hidden": true }
  ],
  "packages": {
    ".": { "release-type": "node" }
  }
}
```

> **`include-component-in-tag: false` is not optional.** Manifest mode defaults
> to `true`, and with `release-type: node` the component comes from the package
> name — so tags become `<plugin>-v1.2.0` instead of `v1.2.0`. The publish
> workflow listens on `v*`, so releases would be created and then never
> published, with no error anywhere. Simple mode does not have this problem,
> which is why it only appears once you switch.

Set `hidden: true` on a type to keep it out of the changelog. Types you omit
entirely are hidden too — list them explicitly so the intent is visible.

`.release-please-manifest.json` — **seed it with the version currently in
`package.json`**, or release-please computes the next release from the wrong
baseline:

```json
{ ".": "0.0.1" }
```

The two modes are mutually exclusive: whenever `release-type` is non-empty the
config file is ignored entirely, so a config that appears to do nothing usually
means `release-type` is still set.

### `publish.yml`

| Input | Default | Purpose |
| --- | --- | --- |
| `ref` | — (required) | Tag to check out and publish |
| `node-version` | `24` | Keep at 24+. Trusted Publishing needs npm >= 11.5.1, and Node 22 still ships npm 10.x |
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

**A release was created but the publish workflow never ran.** Check the tag
name. If it looks like `<plugin>-v1.2.0` rather than `v1.2.0`, you are in
manifest mode without `include-component-in-tag: false`. Fix the config, then
publish the existing tag manually via `workflow_dispatch`.

**`ENEEDAUTH` or 401 on publish.** Usually the wrong workflow filename
registered with npm (it must be the caller in your repo), a caller missing
`id-token: write`, or `node-version` set below 24 (Node 22 ships npm 10.x,
below the 11.5.1 that Trusted Publishing requires).

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

**The first changelog is enormous.** No anchor tag existed. See step 1.
