# Caylent DevContainer Catalog

Specialized development environment configurations for Caylent engineering practices (CAE, CNA, CDE, and Solutions).

This catalog provides curated devcontainer entries that extend the [Caylent DevContainer CLI](https://github.com/caylent-solutions/devcontainer) with practice-specific tooling, dependencies, and configurations.

---

## Table of Contents

1. [What is a DevContainer Catalog?](#1-what-is-a-devcontainer-catalog)
2. [Catalog Repo Structure](#2-catalog-repo-structure)
3. [Creating a New Catalog Repo](#3-creating-a-new-catalog-repo)
4. [Adding a New Entry](#4-adding-a-new-entry)
5. [Modifying Common Assets](#5-modifying-common-assets)
6. [postCreateCommand Reference](#6-postcreatcommand-reference)
7. [Customization Model](#7-customization-model)
8. [Validation Reference](#8-validation-reference)
9. [Distributing Your Catalog](#9-distributing-your-catalog)

---

## 1. What is a DevContainer Catalog?

A catalog is a Git repository containing one or more **entries**. Each entry is a complete devcontainer configuration that can be applied to a project using the Caylent DevContainer CLI (`cdevcontainer`).

Entries share **common assets** (postcreate scripts, shared bash functions, project-setup templates) while providing entry-specific configurations (devcontainer.json, VS Code extensions, features, example files).

**Key concepts:**

- **Catalog** -- A Git repository with a defined directory structure containing common assets and one or more entries.
- **Entry** -- A named directory under `catalog/` that contains a `devcontainer.json`, metadata (`catalog-entry.json`), and a version file (`VERSION`).
- **Common assets** -- Shared scripts in `common/devcontainer-assets/` that are copied into every project regardless of which entry is selected. Common assets take precedence over entry files on name collision.
- **Default catalog** -- The CLI ships with a built-in default catalog URL (`https://github.com/caylent-solutions/devcontainer.git`). Setting `DEVCONTAINER_CATALOG_URL` overrides this with a different catalog (such as this one).

The CLI clones the catalog, presents available entries, and copies the selected entry (plus common assets) into the project's `.devcontainer/` directory.

---

## 2. Catalog Repo Structure

```
caylent-devcontainer-catalog/
  README.md
  CONTRIBUTING.md
  LICENSE
  common/
    devcontainer-assets/
      .devcontainer.postcreate.sh       # Shared postcreate script (runs at container creation)
      devcontainer-functions.sh          # Shared bash functions used by postcreate and project-setup
      project-setup.sh                   # Template for project-specific setup commands
    root-project-assets/                 # Optional: files copied to project root (not .devcontainer/)
      CLAUDE.md                          # AI coding standards for Claude Code
      .claude/                           # Claude Code configuration
        settings.json
        settings.local.json
  catalog/
    default/
      catalog-entry.json                 # Entry metadata (name, description, tags)
      devcontainer.json                  # Devcontainer configuration
      VERSION                            # Semver version string (e.g., 1.0.0)
    cae-terraform/
      catalog-entry.json
      devcontainer.json
      VERSION
      ...                                # Additional entry-specific files
```

### Common Assets (`common/devcontainer-assets/`)

Files in this directory are shared across all entries. When a entry is applied to a project, common assets are copied **after** entry files, meaning common assets overwrite any file with the same name in the entry.

| File | Purpose |
|------|---------|
| `.devcontainer.postcreate.sh` | Main setup script executed by `postCreateCommand`. Handles environment configuration, asdf tool installation, AWS SSO profile setup, git authentication, Claude Code CLI installation, and proxy validation. |
| `devcontainer-functions.sh` | Shared bash functions (logging, WSL compatibility, proxy parsing/validation, git configuration, asdf plugin management, pipx installation). Used by the postcreate script and `project-setup.sh`. |
| `project-setup.sh` | Template script for project-specific setup. Developers customize this file after initial setup to add project-level initialization commands (e.g., `make configure`, `npm install`, database seeds). |

### Root Project Assets (`common/root-project-assets/`) — Optional

If this directory exists, its contents are copied to the **project root** (not `.devcontainer/`) when a entry is applied. This distributes standardized root-level files to all projects.

| File / Directory | Purpose |
|------|---------|
| `CLAUDE.md` | AI coding standards and project instructions for Claude Code. |
| `.claude/settings.json` | Claude Code permission settings (allow/deny lists). |
| `.claude/settings.local.json` | Local Claude Code settings overrides. |

### Entry Files

Each entry directory under `catalog/` must contain these required files:

| File | Required | Description |
|------|----------|-------------|
| `catalog-entry.json` | Yes | Entry metadata: name, description, tags, maintainer, min_cli_version |
| `devcontainer.json` | Yes | The devcontainer configuration (image, features, extensions, postCreateCommand) |
| `VERSION` | Yes | Semver version string (e.g., `1.0.0`) |

Entries may also contain additional files specific to the configuration (e.g., example environment files, proxy toolkit configs, AWS profile map templates). Example files (`example-container-env-values.json`, `example-aws-profile-map.json`) are automatically removed during copy.

### `catalog-entry.json` Schema

```json
{
  "name": "entry-name",
  "description": "Human-readable description of this entry",
  "tags": ["practice", "language", "framework"],
  "maintainer": "Team or individual name",
  "min_cli_version": "2.0.0"
}
```

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `name` | Yes | string | Must match the directory name. Pattern: `^[a-z][a-z0-9-]*[a-z0-9]$` (min 2 chars, lowercase alphanumeric with hyphens, cannot start/end with hyphen). |
| `description` | Yes | string | Displayed when users browse the catalog with `cdevcontainer catalog list`. |
| `tags` | No | array of strings | Used for filtering with `--tags`. Each tag follows the same pattern as `name`. |
| `maintainer` | No | string | Person or team responsible for this entry. |
| `min_cli_version` | No | string | Minimum CLI version required, in semver format `X.Y.Z`. Entries requiring a newer CLI than the user has installed are skipped during listing. |

---

## 3. Creating a New Catalog Repo

To create a new catalog repository from scratch:

### Step 1: Initialize the repository

```bash
mkdir my-devcontainer-catalog
cd my-devcontainer-catalog
git init
```

### Step 2: Create the required directory structure

```bash
mkdir -p common/devcontainer-assets
mkdir -p entries
```

### Step 3: Add the required common assets

The `common/devcontainer-assets/` directory must contain exactly three files:

**`.devcontainer.postcreate.sh`** -- The main postcreate script. This script is executed by `postCreateCommand` in every entry's `devcontainer.json`. It handles:
- Environment variable configuration (sourcing `shell.env` into `.bashrc`/`.zshrc`)
- asdf installation and `.tool-versions` processing
- AWS SSO profile configuration
- Git authentication (token or SSH)
- Package installation (apt, pipx)
- Claude Code CLI installation
- Proxy validation (when `HOST_PROXY=true`)
- Running `project-setup.sh` at the end

```bash
#!/usr/bin/env bash
set -euo pipefail

WORK_DIR=$(pwd)
CONTAINER_USER=$1

# Source shared functions
source "${WORK_DIR}/.devcontainer/devcontainer-functions.sh"

log_info "Starting post-create setup..."
# ... setup logic ...
exit 0
```

**`devcontainer-functions.sh`** -- Shared bash functions:

```bash
#!/usr/bin/env bash

log_info()  { echo -e "\033[0;36m[INFO]\033[0m $1"; }
log_success() { echo -e "\033[0;32m[DONE]\033[0m $1"; }
log_warn()  { echo -e "\033[1;33m[WARN]\033[0m $1"; WARNINGS+=("$1"); }
log_error() { echo -e "\033[0;31m[ERROR]\033[0m $1"; }
exit_with_error() { log_error "$1"; exit 1; }

# ... additional utility functions (is_wsl, install_asdf_plugin,
#     install_with_pipx, parse_proxy_host_port, validate_host_proxy,
#     configure_git_shared, configure_git_token, configure_git_ssh) ...
```

**`project-setup.sh`** -- A minimal template that developers customize per project:

```bash
#!/usr/bin/env bash
set -euo pipefail

source "$(dirname "$0")/devcontainer-functions.sh"

log_info "Running project-specific setup..."

# Add your project setup commands below this line
# Example:
# if [ -f "Makefile" ]; then
#   log_info "Running make configure..."
#   make configure
# fi

log_info "Project-specific setup complete"
```

### Step 4: Add at least one entry

See [Adding a New Entry](#4-adding-a-new-entry).

### Step 5: Validate and push

```bash
pip install caylent-devcontainer-cli
cdevcontainer catalog validate --local .
git add .
git commit -m "Initial catalog structure"
git remote add origin <your-repo-url>
git push -u origin main
```

---

## 4. Adding a New Entry

### Step 1: Create the entry directory

```bash
mkdir -p catalog/my-entry
```

The directory name must match the `name` field in `catalog-entry.json`. It must follow the naming pattern: lowercase, alphanumeric with hyphens, minimum 2 characters, cannot start or end with a hyphen.

Valid names: `default`, `cae-terraform`, `cna-java-spring`, `cde-python-data`, `solutions-fullstack`

Invalid names: `My-Entry` (uppercase), `a` (too short), `-bad-` (starts/ends with hyphen), `has spaces` (spaces), `under_score` (underscores)

### Step 2: Create `catalog-entry.json`

```json
{
  "name": "my-entry",
  "description": "A clear description of what this entry provides",
  "tags": ["practice-name", "language", "framework"],
  "maintainer": "Your Name or Team",
  "min_cli_version": "2.0.0"
}
```

### Step 3: Create `devcontainer.json`

The `devcontainer.json` must include a `postCreateCommand` that references `.devcontainer/.devcontainer.postcreate.sh`. See [postCreateCommand Reference](#6-postcreatcommand-reference) for the required format.

Minimal example:

```json
{
  "name": "my-devcontainer",
  "remoteUser": "vscode",
  "image": "mcr.microsoft.com/devcontainers/base:noble",
  "workspaceFolder": "/workspaces/${localWorkspaceFolderBasename}",
  "features": {
    "ghcr.io/devcontainers/features/aws-cli:1": { "version": "latest" },
    "ghcr.io/devcontainers/features/python:1": { "version": "3.14" }
  },
  "postCreateCommand": "bash -c 'exec > >(tee /tmp/devcontainer-setup.log) 2>&1; source shell.env; sudo -E apt-get update && sudo -E apt-get install -y gettext-base jq && sudo -E bash .devcontainer/.devcontainer.postcreate.sh vscode; echo \"Setup complete.\" && exit 0'"
}
```

### Step 4: Create `VERSION`

```
1.0.0
```

Increment according to semver:
- **Patch** (`1.0.1`): Bug fixes, minor config tweaks
- **Minor** (`1.1.0`): New features, added tools or extensions
- **Major** (`2.0.0`): Breaking changes

### Step 5: Validate

```bash
cdevcontainer catalog validate --local .
```

### File Conflict Rules

Entries must **not** contain files with the same name as common assets:
- `.devcontainer.postcreate.sh`
- `devcontainer-functions.sh`
- `project-setup.sh`

During the copy process, common assets are copied second and overwrite entry files on name collision. Validation reports these conflicts as errors.

---

## 5. Modifying Common Assets

Common assets in `common/devcontainer-assets/` affect **all entries** in this catalog. Changes must be made carefully.

### `.devcontainer.postcreate.sh`

This is the main setup script that runs during container creation. It is invoked by `postCreateCommand` in every entry's `devcontainer.json`. The script:

1. Sources `devcontainer-functions.sh` for shared utilities
2. Validates required environment variables (`DEFAULT_GIT_BRANCH`, etc.)
3. Configures shell environment (`.bashrc`, `.zshrc`, `.zshenv`)
4. Installs asdf and processes `.tool-versions`
5. Installs Oh My Zsh
6. Configures AWS SSO profiles from `aws-profile-map.json`
7. Installs core apt packages and optional extras (`EXTRA_APT_PACKAGES`)
8. Installs the Caylent DevContainer CLI via pipx
9. Installs Claude Code CLI
10. Validates host proxy connectivity (when enabled)
11. Configures git authentication (token or SSH)
12. Runs `project-setup.sh` as the final step

When modifying this file:
- Test changes against at least one entry locally
- The script runs as root (via `sudo -E`) but switches to the container user for user-space operations
- All failures must use `exit_with_error` (fail-fast, non-zero exit)
- Never use `sleep` or time-based delays; use active readiness detection

### `devcontainer-functions.sh`

Provides shared bash functions used by both the postcreate script and `project-setup.sh`. Key functions include:

| Function | Purpose |
|----------|---------|
| `log_info`, `log_success`, `log_warn`, `log_error` | Color-coded logging |
| `exit_with_error` | Log error and exit with code 1 |
| `is_wsl` | Detect Windows Subsystem for Linux environment |
| `install_asdf_plugin` | Install an asdf plugin with idempotency |
| `install_with_pipx` | Install a Python package via pipx (WSL-aware) |
| `parse_proxy_host_port` | Extract host and port from a proxy URL |
| `validate_host_proxy` | Poll proxy reachability with configurable timeout (no sleep) |
| `configure_git_shared` | Write shared `.gitconfig` entries |
| `configure_git_token` | Configure `.netrc` + credential helper for token auth |
| `configure_git_ssh` | Configure SSH key, known_hosts, and SSH config |
| `write_file_with_wsl_compat` | Write files with WSL sudo compatibility |
| `append_to_file_with_wsl_compat` | Append to files with WSL sudo compatibility |

When adding new functions, follow the existing patterns:
- Use `set -euo pipefail` conventions
- Validate all inputs before use
- Provide WSL compatibility branches where file system operations differ
- Use `exit_with_error` for failures

### `project-setup.sh`

This is the template that developers customize after initial setup. Keep it minimal -- it should provide structure and examples but not impose project-specific logic. The script:

- Sources `devcontainer-functions.sh` for logging
- Runs after the main postcreate script completes
- Is the primary extension point for developers

When modifying:
- Consider backward compatibility with existing projects
- Document any new expected environment variables or conventions

---

## 6. postCreateCommand Reference

Every entry's `devcontainer.json` must include a `postCreateCommand` that invokes the shared postcreate script. Validation checks that the literal string `.devcontainer/.devcontainer.postcreate.sh` appears in the command.

### Standard Format (Non-WSL)

```json
{
  "postCreateCommand": "bash -c 'exec > >(tee /tmp/devcontainer-setup.log) 2>&1; source shell.env; sudo -E apt-get update && sudo -E apt-get install -y gettext-base jq && sudo -E bash .devcontainer/.devcontainer.postcreate.sh vscode; echo \"Setup complete. View logs: cat /tmp/devcontainer-setup.log\" && exit 0'"
}
```

Key requirements:
- Must reference `.devcontainer/.devcontainer.postcreate.sh`
- Typically sources `shell.env` first to load environment variables
- Uses `sudo -E` for elevated execution with environment preservation (`-E` passes the caller's environment to the sudo session)
- Wraps output with `tee` to `/tmp/devcontainer-setup.log` for debugging
- Passes the container username as the first argument (typically `vscode`)

### Format With WSL Support

The default entry includes both WSL and non-WSL branches:

```json
{
  "postCreateCommand": "bash -c 'exec > >(tee /tmp/devcontainer-setup.log) 2>&1; source shell.env; if uname -r | grep -i microsoft > /dev/null; then sudo -E apt-get update && sudo -E apt-get install -y gettext-base jq python3 && find .devcontainer -type f -exec sed -i \"s/\\r$//\" {} + && python3 .devcontainer/fix-line-endings.py && sudo -E apt-get remove -y python3 && sudo -E apt-get autoremove -y && sudo -E bash .devcontainer/.devcontainer.postcreate.sh vscode; else sudo -E apt-get update && sudo -E apt-get install -y gettext-base jq && sudo -E bash .devcontainer/.devcontainer.postcreate.sh vscode; fi && echo \"Setup complete. View logs: cat /tmp/devcontainer-setup.log\" && exit 0'"
}
```

WSL-specific handling includes line ending normalization (`\r` removal) and additional Python installation for the `fix-line-endings.py` script.

### Supported Formats for Validation

The CLI validates three formats for `postCreateCommand`:

1. **String** -- A plain command string:
   ```json
   "postCreateCommand": "bash .devcontainer/.devcontainer.postcreate.sh vscode"
   ```

2. **String with `bash -c` wrapper** -- The recommended production format:
   ```json
   "postCreateCommand": "bash -c '... sudo -E bash .devcontainer/.devcontainer.postcreate.sh vscode ...'"
   ```

3. **Array of strings** -- Each element is joined and checked:
   ```json
   "postCreateCommand": ["bash", "-c", "sudo -E bash .devcontainer/.devcontainer.postcreate.sh vscode"]
   ```

All formats must contain the literal string `.devcontainer/.devcontainer.postcreate.sh` somewhere in the command or its elements.

---

## 7. Customization Model

The devcontainer setup follows a three-layer customization model. Each layer has a distinct scope and audience.

### Layer 1: Catalog Common Assets (`common/devcontainer-assets/`)

**Scope:** All entries in this catalog.
**Audience:** Catalog maintainers (platform team).

Shared infrastructure that every entry inherits. Changes here propagate to all entries. This layer provides:
- The postcreate script (tool installation, environment setup, git auth)
- Shared bash functions (logging, WSL compat, proxy validation)
- The project-setup template

Changes at this layer should be infrequent and well-tested. They affect every project that uses this catalog.

### Layer 2: Developer Templates (`~/.devcontainer-templates/`)

**Scope:** A single developer's machine.
**Audience:** Individual developers.

Personal reusable configurations stored in the developer's home directory. These templates let developers maintain their own preferred setups that extend or override catalog entries. Developer templates are not shared via the catalog -- they are local to the developer's workstation.

### Layer 3: Project-Level (`project-setup.sh`)

**Scope:** A single project repository.
**Audience:** Project developers.

Per-project initialization that runs after the postcreate script completes. After a entry is applied to a project, developers edit `.devcontainer/project-setup.sh` to add project-specific commands such as:

```bash
# Install project dependencies
npm install

# Configure local database
make db-migrate

# Download required data files
aws s3 cp s3://my-bucket/seed-data.sql /tmp/seed-data.sql
```

This file is part of the project repository and is version-controlled alongside the project code.

### Copy Order

When `cdevcontainer setup-devcontainer` applies a entry to a project, files are copied in this order:

1. **Entry files copied first** -- All files from `catalog/<name>/` are copied into the project's `.devcontainer/` directory.
2. **Common assets copied second** -- All files from `common/devcontainer-assets/` are copied, overwriting any entry files with the same name.
3. **Root project assets copied** -- If `common/root-project-assets/` exists, its contents are copied to the **project root** (e.g., `CLAUDE.md`, `.claude/`).
4. **`catalog-entry.json` augmented** -- The copied `catalog-entry.json` receives an additional `catalog_url` field recording which catalog URL was used.
5. **Example files removed** -- `example-container-env-values.json` and `example-aws-profile-map.json` are deleted from the target if present.

This ordering ensures that common assets (postcreate script, shared functions, project-setup template) always come from the catalog's `common/` directory, providing consistency across all entries.

---

## 8. Validation Reference

The `cdevcontainer catalog validate` command performs a comprehensive check of catalog structure and content. All checks must pass for validation to succeed.

### Running Validation

```bash
# Validate a local catalog directory (from a clone of this repo)
cdevcontainer catalog validate --local .

# Validate the remote catalog (clones and validates)
DEVCONTAINER_CATALOG_URL="https://github.com/caylent-solutions/caylent-devcontainer-catalog.git" \
  cdevcontainer catalog validate

# Validate the default catalog
cdevcontainer catalog validate
```

### Checks Performed

**Common assets validation:**

| Check | Description |
|-------|-------------|
| Directory exists | `common/devcontainer-assets/` must exist |
| `.devcontainer.postcreate.sh` present | Required common asset |
| `devcontainer-functions.sh` present | Required common asset |
| `project-setup.sh` present | Required common asset |

**Per-entry validation:**

| Check | Description |
|-------|-------------|
| `catalog-entry.json` exists | Required file in every entry |
| `devcontainer.json` exists | Required file in every entry |
| `VERSION` exists | Required file in every entry |
| `name` field present | `catalog-entry.json` must have a non-empty `name` string |
| `description` field present | `catalog-entry.json` must have a non-empty `description` string |
| `name` matches pattern | Must match `^[a-z][a-z0-9-]*[a-z0-9]$` (min 2 chars, lowercase, alphanumeric + hyphens) |
| `name` matches directory | The `name` field value must equal the entry's directory name |
| Tags match pattern | Each tag in `tags` array must match `^[a-z][a-z0-9-]*[a-z0-9]$` |
| `min_cli_version` format | If present, must be valid semver `X.Y.Z` |
| `maintainer` format | If present, must be a non-empty string |
| No duplicate names | No two entries may have the same `name` |
| `postCreateCommand` valid | Must reference `.devcontainer/.devcontainer.postcreate.sh` |
| No file conflicts | Entry must not contain files named `.devcontainer.postcreate.sh`, `devcontainer-functions.sh`, or `project-setup.sh` |

### Common Validation Errors and Fixes

| Error | Fix |
|-------|-----|
| `Missing required directory: common/devcontainer-assets/` | Create the directory and add required common assets |
| `Missing required common asset: .../<file>` | Add the missing file to `common/devcontainer-assets/` |
| `Missing required file: catalog-entry.json` | Create `catalog-entry.json` in the entry directory |
| `Missing required file: devcontainer.json` | Create `devcontainer.json` in the entry directory |
| `Missing required file: VERSION` | Create a `VERSION` file containing a semver string (e.g., `1.0.0`) |
| `'name' is required and must be a non-empty string` | Add the `name` field to `catalog-entry.json` |
| `'name' must be lowercase, dash-separated, min 2 chars` | Rename to match pattern: lowercase, alphanumeric, hyphens only, min 2 chars |
| `'description' is required and must be a non-empty string` | Add a meaningful `description` to `catalog-entry.json` |
| `Duplicate entry name '<name>'` | Change the `name` field to be unique across all entries |
| `postCreateCommand must call .devcontainer/.devcontainer.postcreate.sh` | Update `devcontainer.json` to include the postcreate script reference |
| `Entry contains '<file>' which conflicts with common/...` | Remove the conflicting file from the entry directory |
| `tags[N] must be lowercase, dash-separated` | Fix the tag value to match the naming pattern |
| `'min_cli_version' must be semver (X.Y.Z)` | Use format like `2.0.0`, not `2.0` or `v2.0.0` |

---

## 9. Distributing Your Catalog

### Setting the Catalog URL

Users consume catalogs by setting the `DEVCONTAINER_CATALOG_URL` environment variable:

```bash
export DEVCONTAINER_CATALOG_URL="https://github.com/caylent-solutions/caylent-devcontainer-catalog.git"
```

This overrides the default catalog URL (`https://github.com/caylent-solutions/devcontainer.git`).

### Supported URL Formats

The CLI supports multiple Git URL formats with optional branch/tag references.

**HTTPS:**

```bash
# Default branch
export DEVCONTAINER_CATALOG_URL="https://github.com/caylent-solutions/caylent-devcontainer-catalog.git"
export DEVCONTAINER_CATALOG_URL="https://github.com/caylent-solutions/caylent-devcontainer-catalog"

# Specific branch or tag
export DEVCONTAINER_CATALOG_URL="https://github.com/caylent-solutions/caylent-devcontainer-catalog.git@v2.0"
export DEVCONTAINER_CATALOG_URL="https://github.com/caylent-solutions/caylent-devcontainer-catalog.git@main"
export DEVCONTAINER_CATALOG_URL="https://github.com/caylent-solutions/caylent-devcontainer-catalog@release-1.0"
```

**SSH:**

```bash
# Default branch
export DEVCONTAINER_CATALOG_URL="git@github.com:caylent-solutions/caylent-devcontainer-catalog.git"

# Specific branch or tag
export DEVCONTAINER_CATALOG_URL="git@github.com:caylent-solutions/caylent-devcontainer-catalog.git@v2.0"
export DEVCONTAINER_CATALOG_URL="git@github.com:caylent-solutions/caylent-devcontainer-catalog.git@main"
```

The `@ref` suffix is appended after the URL. When the URL contains `.git`, the ref delimiter is `@` after `.git`. When there is no `.git` suffix, the last `@` in the URL is used as the delimiter (excluding the SSH user prefix `git@`).

### Authentication

For HTTPS: Ensure a valid token or credential helper is configured (e.g., `gh auth login`, `.netrc`, Git credential manager).

For SSH: Ensure your SSH key is loaded (`ssh-add`) and the host is in `~/.ssh/known_hosts`.

You can test access with:

```bash
git ls-remote <catalog-url>
```

### CLI Commands for Catalog Interaction

**List entries:**

```bash
# List all entries
cdevcontainer catalog list

# Filter by tags (comma-separated, matches ANY tag)
cdevcontainer catalog list --tags java,spring
cdevcontainer catalog list --tags aws,kubernetes
```

**Validate catalog:**

```bash
# Validate a local catalog directory
cdevcontainer catalog validate --local /path/to/catalog

# Validate the remote catalog (clones, validates, cleans up)
cdevcontainer catalog validate
```

**Set up a project with a catalog entry:**

```bash
# Interactive selection from available entries
cdevcontainer setup-devcontainer /path/to/project

# Specify a entry by name
cdevcontainer setup-devcontainer --catalog-entry default /path/to/project
```

### Pinning a Catalog Version

Use the `@ref` syntax to pin a specific branch, tag, or commit to ensure reproducible setups:

```bash
# Pin to a release tag
export DEVCONTAINER_CATALOG_URL="https://github.com/caylent-solutions/caylent-devcontainer-catalog.git@v1.0.0"

# Pin to a stable branch
export DEVCONTAINER_CATALOG_URL="https://github.com/caylent-solutions/caylent-devcontainer-catalog.git@main"
```

This is recommended for CI/CD pipelines and team-wide configuration to avoid unexpected changes from catalog updates.

### Self-Contained Catalogs

This catalog has its own `common/devcontainer-assets/` -- independent of the default catalog. The postcreate script, shared functions, and project-setup template can be customized for Caylent-specific needs without affecting the default catalog or any other catalog.

---

## Related Resources

- [Caylent DevContainer CLI](https://github.com/caylent-solutions/devcontainer) -- The CLI tool that consumes catalogs
- [Default Catalog](https://github.com/caylent-solutions/devcontainer) -- The default general-purpose catalog (same repo as CLI)
- [Contributing Guide](CONTRIBUTING.md) -- How to add and maintain entries in this catalog
