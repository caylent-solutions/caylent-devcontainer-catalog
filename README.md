# Caylent DevContainer Catalog

Specialized development environment configurations for Caylent engineering practices (CAE, CNA, CDE, and Solutions).

This catalog provides curated devcontainer collections that extend the [Caylent DevContainer CLI](https://github.com/caylent-solutions/devcontainer) with practice-specific tooling, dependencies, and configurations.

## What is a Catalog?

A catalog is a Git repository containing one or more **collections** — each collection is a complete devcontainer configuration that can be applied to a project. Collections share common assets (postcreate scripts, shared functions) while providing collection-specific configurations (devcontainer.json, proxy toolkits, example files).

## Repository Structure

```
caylent-devcontainer-catalog/
  README.md                             # This file
  CONTRIBUTING.md                       # How to add and maintain collections
  LICENSE                               # Apache 2.0
  common/
    devcontainer-assets/
      .devcontainer.postcreate.sh       # Shared postcreate script (runs at container creation)
      devcontainer-functions.sh         # Shared bash functions used by postcreate and project-setup
      project-setup.sh                  # Template for project-specific setup commands
  collections/
    <collection-name>/
      catalog-entry.json                # Collection metadata (name, description, tags)
      devcontainer.json                 # Devcontainer configuration
      VERSION                           # Semver version string
      ...                               # Additional collection-specific files
```

### Common Assets

Files in `common/devcontainer-assets/` are shared across all collections. When a collection is applied to a project, common assets are copied **after** collection files, meaning common assets take precedence over any conflicting files in the collection directory.

| File | Purpose |
|------|---------|
| `.devcontainer.postcreate.sh` | Main setup script executed by `postCreateCommand` in devcontainer.json. Handles environment configuration, tool installation, AWS setup, git authentication, and more. |
| `devcontainer-functions.sh` | Shared bash functions (logging, WSL compatibility, proxy validation, git configuration) used by the postcreate script and project-setup.sh. |
| `project-setup.sh` | Template script for project-specific setup. Developers customize this file after initial setup to add project-level initialization commands. |

### Collections

Each collection under `collections/` represents a distinct devcontainer configuration. A collection must contain:

| File | Required | Description |
|------|----------|-------------|
| `catalog-entry.json` | Yes | Collection metadata: `name`, `description`, `tags`, `maintainer`, `min_cli_version` |
| `devcontainer.json` | Yes | The devcontainer configuration file |
| `VERSION` | Yes | Semver version string (e.g., `1.0.0`) |

#### catalog-entry.json Schema

```json
{
  "name": "collection-name",
  "description": "Human-readable description of this collection",
  "tags": ["practice", "language", "framework"],
  "maintainer": "Team or individual name",
  "min_cli_version": "2.0.0"
}
```

- **name**: Must match the directory name. Lowercase, alphanumeric, hyphens only.
- **description**: Required. Displayed when users browse the catalog.
- **tags**: Optional. Used for filtering with `cdevcontainer catalog list --tags`.
- **maintainer**: Optional. Who maintains this collection.
- **min_cli_version**: Optional. Minimum CLI version required.

## Usage

### Setting Up a Project with this Catalog

Set the `DEVCONTAINER_CATALOG_URL` environment variable to point to this catalog:

```bash
export DEVCONTAINER_CATALOG_URL="https://github.com/caylent-solutions/caylent-devcontainer-catalog.git"
```

Then run the setup command:

```bash
cdevcontainer setup-devcontainer /path/to/project
```

The CLI will present available collections from this catalog for selection.

### Using a Specific Collection

```bash
cdevcontainer setup-devcontainer --catalog-entry <collection-name> /path/to/project
```

### Browsing Available Collections

```bash
# List all collections in this catalog
DEVCONTAINER_CATALOG_URL="https://github.com/caylent-solutions/caylent-devcontainer-catalog.git" \
  cdevcontainer catalog list

# Filter by tags
DEVCONTAINER_CATALOG_URL="https://github.com/caylent-solutions/caylent-devcontainer-catalog.git" \
  cdevcontainer catalog list --tags java
```

### Validating the Catalog

```bash
# Validate locally (from a clone of this repo)
cdevcontainer catalog validate --local .

# Validate remotely
DEVCONTAINER_CATALOG_URL="https://github.com/caylent-solutions/caylent-devcontainer-catalog.git" \
  cdevcontainer catalog validate
```

## Customization Model

The devcontainer setup follows a three-layer customization model:

1. **Common assets** (this catalog's `common/devcontainer-assets/`) — Shared infrastructure that all collections inherit. Changes here affect all collections in this catalog.

2. **Collection files** (e.g., `collections/my-collection/`) — Collection-specific configuration. Provides the devcontainer.json, metadata, and any collection-specific files (proxy configs, example files, etc.).

3. **Project-level customization** (`project-setup.sh`) — After a collection is applied to a project, the developer edits `.devcontainer/project-setup.sh` to add project-specific initialization commands. This file is overwritten on re-setup, so developers should merge back customizations after upgrades.

## Self-Contained Catalog

This catalog has its own `common/devcontainer-assets/` — independent of the [default catalog](https://github.com/caylent-solutions/devcontainer). The postcreate script, shared functions, and project-setup template can be customized for Caylent-specific needs without affecting the default catalog or any other catalog.

## Related Resources

- [Caylent DevContainer CLI](https://github.com/caylent-solutions/devcontainer) — The CLI tool that consumes catalogs
- [Default Catalog](https://github.com/caylent-solutions/devcontainer) — The default general-purpose catalog (same repo as CLI)
