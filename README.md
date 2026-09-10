# GCDS Plugins

Welcome to the GCDS Extensions plugin ecosystem.

This repository holds the shared publishing workflows and documentation for the
GCDS Extensions plugin ecosystem. Plugins live in their own repositories; this one is what
they inherit from.

Plugins extend the Government of Canada Design System (GCDS) with optional
components maintained outside the core library, so teams can build and
distribute extra functionality without adding weight to GCDS itself.

## What is a plugin?

A plugin is an optional component that extends GCDS without increasing the size
or maintenance burden of the core library.

Each plugin:

* Is published as its own npm package under the `@gcds-extensions` scope.
* Registers one or more custom elements using the `gcds-ext-*` prefix.
* Lives in its own repository in the `gcds-extensions` GitHub org.
* Is maintained by its own owners.
* Releases through the shared workflows in this repository.

For example:

| Repository                     | npm package                     | Custom element            |
| ------------------------------ | ------------------------------- | ------------------------- |
| `gcds-extensions/code-display` | `@gcds-extensions/code-display` | `<gcds-ext-code-display>` |

## Publishing, at a glance

Releases are driven by [conventional commits](https://www.conventionalcommits.org/)
and run through two shared workflows:

```
commit to main ──> release-generator ──> opens a release PR
                                             │ you merge it
                                             ▼
                                  version bumped, CHANGELOG written,
                                       tag vX.Y.Z created
                                             │
                                             ▼
                                       publish ──> npm
```

You never set the version by hand, and merging the release PR is what ships the
package. Your plugin repo holds two small caller files; the logic lives here, so
a fix here reaches every plugin.

| Workflow | What it does |
| --- | --- |
| [`.github/workflows/release-generator.yml`](./.github/workflows/release-generator.yml) | Maintains the release PR, then tags and creates the GitHub release when it is merged |
| [`.github/workflows/publish.yml`](./.github/workflows/publish.yml) | Builds the tagged commit and publishes it to npm via [Trusted Publishing](https://docs.npmjs.com/trusted-publishers) (OIDC) |

Publishing uses [npm Trusted Publishing](https://docs.npmjs.com/trusted-publishers), so no npm tokens are stored in any
plugin repository.

> [!NOTE]
> Before Trusted Publishing can be set up, a GCDS team member publishes an empty placeholder version of your package to NPM at `0.0.0`. NPM can only attach a Trusted Publisher to a package that already exists, so this one-time manual publish has to happen first. You don't need to do anything — once it's done, every later release publishes automatically with no NPM token.

## Getting started

Setting up a new plugin for release takes about fifteen minutes and needs a
GitHub App credential from the GCDS team plus a one-time npm bootstrap publish.

* **[Publishing a plugin](./docs/publishing.md)** — start here. Prerequisites,
  seven setup steps, day-to-day use, full workflow reference, and troubleshooting.
* **[Plugin directory](./plugin-directory.md)** — what exists and who maintains it.

Not yet written: guides for creating a plugin, plugin requirements, and
maintaining a plugin. Until they land, the publishing guide is the authoritative
reference for anything release-related.

## Repository contents

```
.github/workflows/
  release-generator.yml   reusable — release PRs, tagging, GitHub releases
  publish.yml             reusable — build and publish to npm
docs/
  publishing.md           how to set up and use the workflows
plugin-directory.md       the list of plugins and their maintainers
```

## Related repositories

Core GCDS packages:

* `@gcds-core/components`
* `@gcds-core/components-react`
* `@gcds-core/components-vue`
* `@gcds-core/components-angular`

> [!IMPORTANT]
> Not every package published under the `@gcds-extensions` npm scope is a plugin. Some packages provide implementation-specific integrations or other ecosystem functionality and may follow different contribution and maintenance models.
