# Plugin Directory

This page lists all available GCDS plugins.

Plugins are optional components that extend GCDS. Each plugin is independently versioned, published, and maintained by its respective owners.

## Available plugins

| Plugin       | Repository                     | npm package                     | Custom element            | Maintainer | Status  |
| ------------ | ------------------------------ | ------------------------------- | ------------------------- | ---------- | ------- |
| Code Display | `gcds-extensions/code-display` | `@gcds-extensions/code-display` | `<gcds-ext-code-display>` | GCDS       | Planned |
| Map          | *gcds-extensions/map*          | `@gcds-extensions/map`          | `<gcds-ext-map>`          | NRCAN      | Planned |

## Plugin lifecycle

| Status      | Description                                                                                                       |
| ----------- | ----------------------------------------------------------------------------------------------------------------- |
| Planned     | The plugin has been proposed or is currently under development.                                                   |
| Active      | The plugin is available for use and actively maintained.                                                          |
| Deprecated  | The plugin should not be adopted for new projects and may be removed in a future release.                         |

## Maintainers

Every plugin has an identified maintainer responsible for:

* Triaging issues and feature requests.
* Maintaining compatibility with supported GCDS releases.
* Publishing new releases.
* Keeping documentation up to date.
* Responding to community questions.

Maintainers may be the GCDS team, another Government of Canada team, or external contributors.

## Looking for another package?

Not every package published under the `@gcds-extensions` npm scope is a plugin. Some packages provide implementation-specific integrations or other ecosystem functionality and may follow different contribution and maintenance models.
