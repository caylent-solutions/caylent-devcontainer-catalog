# Contributing to the Caylent DevContainer Catalog

This guide covers how to add new collections, maintain existing ones, and contribute to the shared common assets.

## Table of Contents

- [Adding a New Collection](#adding-a-new-collection)
- [Collection Naming Conventions](#collection-naming-conventions)
- [Required Files](#required-files)
- [Validation](#validation)
- [Modifying Common Assets](#modifying-common-assets)
- [Pull Request Process](#pull-request-process)
- [Testing](#testing)

## Adding a New Collection

1. Create a directory under `collections/` with a descriptive name:

   ```bash
   mkdir -p collections/my-collection
   ```

2. Create the required files (see [Required Files](#required-files) below).

3. Validate the catalog:

   ```bash
   cdevcontainer catalog validate --local .
   ```

4. Open a pull request.

## Collection Naming Conventions

Collection directory names and the `name` field in `catalog-entry.json` must:

- Be lowercase
- Use hyphens (`-`) as word separators
- Be alphanumeric (plus hyphens)
- Match the directory name exactly
- Be descriptive of the practice or technology stack

Examples:
- `cae-terraform` — CAE practice with Terraform focus
- `cna-java-spring` — CNA practice with Java Spring Boot
- `cde-python-data` — CDE practice with Python data engineering tools
- `solutions-fullstack` — Solutions practice full-stack configuration

## Required Files

Every collection must contain these files:

### catalog-entry.json

```json
{
  "name": "my-collection",
  "description": "A clear description of what this collection provides",
  "tags": ["practice-name", "language", "framework"],
  "maintainer": "Your Name or Team",
  "min_cli_version": "2.0.0"
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Must match the directory name |
| `description` | Yes | Human-readable description shown in catalog list |
| `tags` | No | Array of strings for filtering (`cdevcontainer catalog list --tags`) |
| `maintainer` | No | Person or team responsible for this collection |
| `min_cli_version` | No | Minimum CLI version that supports this collection |

### devcontainer.json

The devcontainer configuration. Must include a `postCreateCommand` that invokes the shared postcreate script:

```json
{
  "postCreateCommand": "bash .devcontainer/.devcontainer.postcreate.sh ${localEnv:USER:vscode}"
}
```

### VERSION

A file containing a semver version string (e.g., `1.0.0`). Increment this when making changes to the collection:

- **Patch** (`1.0.1`): Bug fixes, minor config tweaks
- **Minor** (`1.1.0`): New features, added tools
- **Major** (`2.0.0`): Breaking changes to the collection

## File Conflict Rules

Collections must **not** contain files with the same name as common assets:

- `.devcontainer.postcreate.sh`
- `devcontainer-functions.sh`
- `project-setup.sh`

If a collection contains these files, the common assets version will take precedence during copy. Validation will warn about conflicts.

## Validation

Before submitting a PR, validate the catalog structure:

```bash
# Install the CLI if not already installed
pip install caylent-devcontainer-cli

# Validate the catalog
cdevcontainer catalog validate --local .
```

Validation checks:
- `common/devcontainer-assets/` exists with all required files
- Each collection has `catalog-entry.json`, `devcontainer.json`, and `VERSION`
- `catalog-entry.json` contains required fields (`name`, `description`)
- `name` in `catalog-entry.json` matches the directory name
- No duplicate collection names across the catalog
- `devcontainer.json` has a valid `postCreateCommand` referencing the postcreate script
- No file conflicts between collections and common assets

## Modifying Common Assets

Common assets in `common/devcontainer-assets/` affect **all collections** in this catalog. Changes should be made carefully:

1. **`.devcontainer.postcreate.sh`** — The main setup script. Handles environment setup, tool installation, AWS configuration, git authentication. Modify this to change what happens for all collections during container creation.

2. **`devcontainer-functions.sh`** — Shared bash functions. Add new utility functions here that can be used by the postcreate script or project-setup.sh.

3. **`project-setup.sh`** — The template that developers customize. Keep this minimal — it should provide examples and structure but not impose project-specific logic.

When modifying common assets:
- Test changes against at least one collection
- Document the change in the PR description
- Consider backward compatibility with existing projects using this catalog

## Pull Request Process

1. **Branch**: Create a feature branch from `main`
2. **Changes**: Make your changes (new collection, common asset update, etc.)
3. **Validate**: Run `cdevcontainer catalog validate --local .`
4. **PR**: Open a pull request with:
   - Description of what changed and why
   - For new collections: what practice/team this serves, what tools are included
   - For common asset changes: impact assessment on existing collections
5. **Review**: At least 1 approving review required
6. **Merge**: Squash merge only

### PR Checklist

- [ ] `cdevcontainer catalog validate --local .` passes
- [ ] Collection `name` matches directory name
- [ ] `catalog-entry.json` has meaningful `description` and `tags`
- [ ] `devcontainer.json` references the shared postcreate script
- [ ] `VERSION` file is present and contains a valid semver string
- [ ] No file conflicts with common assets
- [ ] PR description explains the purpose and impact

## Testing

### Local Testing

To test a collection locally before pushing:

```bash
# 1. Clone this repo
git clone https://github.com/caylent-solutions/caylent-devcontainer-catalog.git
cd caylent-devcontainer-catalog

# 2. Validate structure
cdevcontainer catalog validate --local .

# 3. Test with a project
mkdir -p /tmp/test-project
DEVCONTAINER_CATALOG_URL="/path/to/caylent-devcontainer-catalog" \
  cdevcontainer setup-devcontainer /tmp/test-project

# 4. Verify the devcontainer builds and works
cd /tmp/test-project
code .
# Accept "Reopen in Container" prompt
```

### Validation Errors

Common validation errors and fixes:

| Error | Fix |
|-------|-----|
| `Missing required common asset: <file>` | Add the missing file to `common/devcontainer-assets/` |
| `Missing required file: catalog-entry.json` | Create `catalog-entry.json` in your collection directory |
| `Missing required field: name` | Add the `name` field to `catalog-entry.json` |
| `Duplicate collection name: <name>` | Change the `name` field to be unique across all collections |
| `postCreateCommand does not reference postcreate script` | Update `devcontainer.json` to include the postcreate command |
