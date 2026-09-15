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

* Is published as its own NPM package under the `@gcds-extensions` scope.
* Registers one or more custom elements using the `gcds-ext-*` prefix.
* Lives in its own repository in the `gcds-extensions` GitHub org.
* Is maintained by its own owners.
* Releases through the shared workflows in this repository.

Names follow one pattern throughout:

```
repository        gcds-extensions/<name>
NPM package       @gcds-extensions/<name>
custom element    <gcds-ext-<name>>
```

## Available plugins

| Plugin | NPM package | What it does | Maintainer | Status |
| --- | --- | --- | --- | --- |
| **Map** | `@gcds-extensions/map` | _TODO: one line_ | NRCan | Planned |
| **Code Display** | `@gcds-extensions/code-display` | **Internal.** Built for the GC Design System's own documentation, to render live code examples. Published openly, but not aimed at general use. | GCDS | Planned |

Each plugin's repository and custom element follow the naming pattern above.
The [plugin directory](./plugin-directory.md) has the status definitions and
what maintainers are responsible for.

From here, two paths: [already building a plugin](#already-building-a-plugin),
or [thinking about a new one](#thinking-about-a-new-plugin).

## Already building a plugin

Plugins don't carry release logic of their own — they point at the shared
workflows in this repository, so a fix made once lands in every plugin.

### How releases work

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
                                       publish ──> NPM
```

You never set the version by hand, and merging the release PR is what ships the
package.

| Workflow | What it does |
| --- | --- |
| [`.github/workflows/release-generator.yml`](./.github/workflows/release-generator.yml) | Maintains the release PR, then tags and creates the GitHub release when it is merged |
| [`.github/workflows/publish.yml`](./.github/workflows/publish.yml) | Builds the tagged commit and publishes it to NPM via [Trusted Publishing](https://docs.npmjs.com/trusted-publishers) (OIDC) |

Publishing uses [NPM Trusted Publishing](https://docs.npmjs.com/trusted-publishers), so no NPM tokens are stored in any
plugin repository.

> [!NOTE]
> Before Trusted Publishing can be set up, a GCDS team member publishes an empty placeholder version of your package to NPM at `0.0.0`. NPM can only attach a Trusted Publisher to a package that already exists, so this one-time manual publish has to happen first. You don't need to do anything — once it's done, every later release publishes automatically with no NPM token.

### Setting it up

Copy two workflow files into your repo. On the GitHub side a GCDS org owner
adds your repo to the release bot App and to its two org secrets — no
credentials are handed to you. On the NPM side the package needs a one-time
bootstrap publish and a Trusted Publisher entry, because both are configured
per package.

* **[Publishing a plugin](./docs/publishing.md)** — start here. Prerequisites,
  seven setup steps, day-to-day use, full workflow reference, and troubleshooting.
* **[Plugin directory](./plugin-directory.md)** — what exists and who maintains it.

Written guides for plugin requirements and ongoing maintenance are still to
come. Until they land the team walks you through those directly, and the
publishing guide is the authoritative reference for anything release-related.

## Thinking about a new plugin

Plugins are not self-serve. Every plugin is published under the
`@gcds-extensions` scope and carries the design system's name, so we agree on
scope and ownership before a repository exists. An unmaintained plugin is worse
for teams than no plugin at all.

Get in touch when you have:

* a specific plugin in mind, and
* approval from your department to own and maintain it long term

**[Contact the GC Design System team](https://design-system.canada.ca/en/contact/)**
and tell us what you are proposing. We will work through scope, naming, and
repository setup with you, and take care of the org-level pieces — the release
bot, the NPM scope, and Trusted Publishing.

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
> Not every package published under the `@gcds-extensions` NPM scope is a plugin. Some packages provide implementation-specific integrations or other ecosystem functionality and may follow different contribution and maintenance models.
