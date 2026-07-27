# GCDS Plugins

Welcome to the GCDS plugin ecosystem.

This repository contains the documentation, tooling, and workflows for creating, publishing, and maintaining GCDS plugins.

Plugins extend the Government of Canada Design System (GCDS) with optional components that are maintained outside of the core GCDS library. They allow teams to build and distribute additional functionality while following a consistent set of patterns and contribution practices.

## What is a plugin?

A plugin is an optional component that extends GCDS without increasing the size or maintenance burden of the core library.

Each plugin:

* Is published as its own npm package.
* Registers one or more custom elements using the `gcds-ext-*` prefix.
* Is maintained by its own owners.
* Follows the conventions documented in this repository.

For example:

| Repository                     | npm package                     | Custom element            |
| ------------------------------ | ------------------------------- | ------------------------- |
| `gcds-extensions/code-display` | `@gcds-extensions/code-display` | `<gcds-ext-code-display>` |

## Getting started

Whether you're creating a new plugin or contributing to an existing one, start with:

* [Plugin Directory](./plugin-directory.md)
* Creating a Plugin
* Plugin Requirements
* [Publishing a Plugin](./docs/publishing.md)
* Maintaining a Plugin

## Publishing a plugin

Plugins publish through the shared reusable workflow in this repository. For full setup and examples (plugin `release.yml`, `package.json` release scripts, bootstrap `0.0.0-alpha` publish, and npm Trusted Publishing configuration), see [docs/publishing.md](./docs/publishing.md).

## Repository contents

This repository contains:

* Plugin documentation
* Publishing workflows
* Contribution guidance
* Templates and examples
* Shared tooling for plugin repositories

## Related repositories

Core GCDS packages:

* `@gcds-core/components`
* `@gcds-core/components-react`
* `@gcds-core/components-vue`
* `@gcds-core/components-angular`

> **Note**
>
> Not every package published under the `@gcds-extensions` npm scope is a plugin. Some packages provide implementation-specific integrations or other ecosystem functionality and may follow different contribution and maintenance models.
