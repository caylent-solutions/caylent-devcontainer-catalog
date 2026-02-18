# Contributing to the Caylent DevContainer Catalog

This guide covers how to add new entries, maintain existing ones, and contribute to the shared common assets in the `caylent-solutions/caylent-devcontainer-catalog` repository.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Entry Naming Conventions](#entry-naming-conventions)
- [Required Files](#required-files)
- [File Conflict Rules](#file-conflict-rules)
- [Adding a New Entry](#adding-a-new-entry)
- [Validation](#validation)
  - [Running Validation](#running-validation)
  - [What Validation Checks](#what-validation-checks)
  - [Common Validation Errors and Fixes](#common-validation-errors-and-fixes)
- [Testing](#testing)
  - [Local Testing Workflow](#local-testing-workflow)
  - [Verifying the DevContainer Build](#verifying-the-devcontainer-build)
- [Modifying Common Assets](#modifying-common-assets)
- [Pull Request Process](#pull-request-process)
  - [Branch and Merge Strategy](#branch-and-merge-strategy)
  - [PR Requirements](#pr-requirements)
  - [Review Checklist](#review-checklist)

---

## Prerequisites

Install the Caylent DevContainer CLI before contributing:

```bash
pip install caylent-devcontainer-cli
```

Verify the installation:

```bash
cdevcontainer --version
```

---

## Entry Naming Conventions

Every entry directory name and the `name` field in its `catalog-entry.json` must follow these rules:

| Rule | Detail |
|------|--------|
| **Pattern** | Must match `^[a-z][a-z0-9-]*[a-z0-9]$` |
| **Minimum length** | 2 characters |
| **Allowed characters** | Lowercase letters (`a-z`), digits (`0-9`), and hyphens (`-`) |
| **Must start with** | A lowercase letter |
| **Must end with** | A lowercase letter or digit |
| **Must not start or end with** | A hyphen |
| **Directory match** | The `name` field in `catalog-entry.json` must match the directory name exactly |
| **Descriptive** | Should describe the practice area, technology stack, or both |

Tags in `catalog-entry.json` follow the same pattern rules as names.

**Examples of valid entry names:**

| Name | Description |
|------|-------------|
| `cae-terraform` | CAE practice with Terraform focus |
| `cna-java-spring` | CNA practice with Java Spring Boot |
| `cde-python-data` | CDE practice with Python data engineering tools |
| `solutions-fullstack` | Solutions practice full-stack configuration |

**Examples of invalid entry names:**

| Name | Reason |
|------|--------|
| `MyEntry` | Uppercase letters not allowed |
| `-terraform` | Cannot start with a hyphen |
| `terraform-` | Cannot end with a hyphen |
| `a` | Must be at least 2 characters |
| `cae_terraform` | Underscores not allowed; use hyphens |
| `cae terraform` | Spaces not allowed |

---

## Required Files

Every entry directory under `catalog/` must contain exactly three files.

### catalog-entry.json

Metadata describing the entry. Used by the CLI when listing, filtering, and selecting entries.

```json
{
  "name": "cna-java-spring",
  "description": "CNA practice environment with Java 17, Spring Boot 3.x, Gradle, and Kubernetes tooling",
  "tags": ["cna", "java", "spring-boot", "kubernetes"],
  "maintainer": "CNA Platform Team",
  "min_cli_version": "2.0.0"
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `name` | **Yes** | Must match the entry directory name exactly. Must follow the naming pattern. |
| `description` | **Yes** | Human-readable description displayed when users browse the catalog. |
| `tags` | No | Array of strings for filtering with `cdevcontainer catalog list --tags`. Each tag must follow the same naming pattern as entry names. |
| `maintainer` | No | Person or team responsible for maintaining this entry. |
| `min_cli_version` | No | Minimum CLI version required to use this entry (semver string). |

### devcontainer.json

The devcontainer configuration file. This file defines the container image, features, extensions, settings, and lifecycle commands for the development environment.

The `postCreateCommand` field is required and must reference the shared postcreate script:

```json
{
  "postCreateCommand": "bash .devcontainer/.devcontainer.postcreate.sh ${localEnv:USER:vscode}"
}
```

The exact command can vary (for example, it may include logging or WSL handling), but it must contain a reference to `.devcontainer/.devcontainer.postcreate.sh`. Validation checks for this reference.

### VERSION

A plain text file containing a single semver version string:

```
1.0.0
```

Follow semantic versioning when updating:

| Change Type | Version Bump | Examples |
|-------------|--------------|----------|
| Bug fixes, minor config tweaks | Patch (`1.0.0` -> `1.0.1`) | Fix a typo in settings, adjust a port number |
| New features, added tools | Minor (`1.0.0` -> `1.1.0`) | Add a new VS Code extension, add a devcontainer feature |
| Breaking changes | Major (`1.0.0` -> `2.0.0`) | Change the base image, remove a tool, restructure the entry |

---

## File Conflict Rules

Entries must **not** contain files with the same names as the common assets. The following filenames are reserved:

- `.devcontainer.postcreate.sh`
- `devcontainer-functions.sh`
- `project-setup.sh`

These files live in `common/devcontainer-assets/` and are copied into every project **after** entry files during setup. If a entry contains a file with one of these names, the common asset version will overwrite it. Validation flags this as a conflict.

If your entry needs custom behavior provided by these scripts, contribute the changes to the common assets instead (see [Modifying Common Assets](#modifying-common-assets)).

---

## Adding a New Entry

1. Create a directory under `catalog/` with a name that follows the [naming conventions](#entry-naming-conventions):

   ```bash
   mkdir -p catalog/cna-java-spring
   ```

2. Create the three required files:

   ```bash
   # catalog-entry.json
   cat > catalog/cna-java-spring/catalog-entry.json << 'EOF'
   {
     "name": "cna-java-spring",
     "description": "CNA practice environment with Java 17, Spring Boot 3.x, Gradle, and Kubernetes tooling",
     "tags": ["cna", "java", "spring-boot"],
     "maintainer": "CNA Platform Team",
     "min_cli_version": "2.0.0"
   }
   EOF

   # VERSION
   echo "1.0.0" > catalog/cna-java-spring/VERSION

   # devcontainer.json (customize features and extensions for your stack)
   cat > catalog/cna-java-spring/devcontainer.json << 'EOF'
   {
     "name": "cna-java-spring",
     "image": "mcr.microsoft.com/devcontainers/java:17",
     "features": {
       "ghcr.io/devcontainers/features/aws-cli:1": { "version": "latest" }
     },
     "postCreateCommand": "bash .devcontainer/.devcontainer.postcreate.sh ${localEnv:USER:vscode}"
   }
   EOF
   ```

3. Add any additional entry-specific files (proxy configurations, example files, etc.) to the same directory. Do not add files that conflict with common assets.

4. Validate the catalog:

   ```bash
   cdevcontainer catalog validate --local .
   ```

5. Test the entry locally (see [Testing](#testing)).

6. Open a pull request (see [Pull Request Process](#pull-request-process)).

---

## Validation

### Running Validation

From the root of the catalog repository:

```bash
cdevcontainer catalog validate --local .
```

Run this command before every commit and before opening a pull request. The command exits with a non-zero status if any checks fail.

### What Validation Checks

The validation command performs the following checks:

**Common asset checks:**
- `common/devcontainer-assets/` directory exists
- All four required common asset files are present:
  - `.devcontainer.postcreate.sh`
  - `devcontainer-functions.sh`
  - `postcreate-wrapper.sh`
  - `project-setup.sh`
- Subdirectories `nix-family-os/` and `wsl-family-os/` exist with required files (`README.md`, `tinyproxy.conf.template`, `tinyproxy-daemon.sh`)
- Shell scripts have executable permission (`.devcontainer.postcreate.sh`, `devcontainer-functions.sh`, `postcreate-wrapper.sh`, `project-setup.sh`, `tinyproxy-daemon.sh`)
- All `.json` files in `common/root-project-assets/` (if present) are valid JSON

**Per-entry checks:**
- `catalog-entry.json` exists in the entry directory
- `devcontainer.json` exists in the entry directory
- `VERSION` exists in the entry directory and contains valid semver (`X.Y.Z`)
- `catalog-entry.json` contains the required `name` field
- `catalog-entry.json` contains the required `description` field
- `catalog-entry.json` contains only known fields (`name`, `description`, `tags`, `maintainer`, `min_cli_version`)
- The `name` field matches the entry directory name
- The `name` field matches the naming pattern (`^[a-z][a-z0-9-]*[a-z0-9]$`, minimum 2 characters)
- All `tags` values match the naming pattern
- No duplicate entry names across all entries in the catalog
- `devcontainer.json` has a non-empty `name` field
- `devcontainer.json` has at least one container source field (`image`, `build`, `dockerFile`, `dockerComposeFile`)
- `devcontainer.json` contains a `postCreateCommand` that references `.devcontainer/.devcontainer.postcreate.sh`
- No file or directory name conflicts between entry files and common assets (including subdirectories)

### Common Validation Errors and Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| Missing required common asset: `<file>` | A file is missing from `common/devcontainer-assets/` | Add the missing file to `common/devcontainer-assets/` |
| Missing subdirectory: `<subdir>` | A required subdirectory is missing from common assets | Create the subdirectory (`nix-family-os/` or `wsl-family-os/`) |
| Missing required file in `<subdir>`: `<file>` | A required file is missing from a subdirectory | Add the missing file to the subdirectory |
| `<file>` is not executable | A shell script lacks executable permission | Run `chmod +x` on the script |
| Invalid JSON in root-project-assets: `<file>` | A `.json` file has invalid syntax | Fix the JSON syntax in the file |
| Missing required file: `catalog-entry.json` | Entry directory lacks its metadata file | Create `catalog-entry.json` in the entry directory with `name` and `description` fields |
| Missing required file: `devcontainer.json` | Entry directory lacks its devcontainer configuration | Create `devcontainer.json` with `name`, container source, and `postCreateCommand` |
| Missing required file: `VERSION` | Entry directory lacks its version file | Create a `VERSION` file containing a semver string (e.g., `1.0.0`) |
| VERSION must contain valid semver | `VERSION` file has invalid format | Ensure it contains only `X.Y.Z` (e.g., `1.0.0`) |
| Missing required field: `name` | `catalog-entry.json` does not include `name` | Add a `name` field that matches the directory name |
| Missing required field: `description` | `catalog-entry.json` does not include `description` | Add a `description` field with a human-readable description |
| Unknown fields in `catalog-entry.json` | Unrecognized fields present | Remove fields not in: `name`, `description`, `tags`, `maintainer`, `min_cli_version` |
| Name does not match directory | `name` in `catalog-entry.json` differs from the directory name | Change the `name` field to match the directory name exactly |
| Name does not match pattern | `name` or `tag` contains invalid characters | Use only lowercase letters, digits, and hyphens. Must start with a letter, end with a letter or digit, minimum 2 characters. |
| devcontainer.json must have a `name` field | `devcontainer.json` is missing the `name` field | Add a non-empty `name` field to `devcontainer.json` |
| devcontainer.json must have a container source | No container source field present | Add `image`, `build`, `dockerFile`, or `dockerComposeFile` to `devcontainer.json` |
| Duplicate entry name: `<name>` | Two or more entries share the same `name` | Change the `name` field (and directory) of one entry to be unique |
| postCreateCommand does not reference postcreate script | `devcontainer.json` is missing the postcreate reference | Add or update `postCreateCommand` to include `bash .devcontainer/.devcontainer.postcreate.sh` |
| File conflict with common asset: `<file>` | Entry contains a file that shares a name with a common asset | Remove the conflicting file from the entry. Contribute needed changes to `common/devcontainer-assets/` instead. |

---

## Testing

### Local Testing Workflow

1. **Clone the repository:**

   ```bash
   git clone https://github.com/caylent-solutions/caylent-devcontainer-catalog.git
   cd caylent-devcontainer-catalog
   ```

2. **Run validation:**

   ```bash
   cdevcontainer catalog validate --local .
   ```

   Fix any errors before proceeding.

3. **Test the entry with a project:**

   Point `DEVCONTAINER_CATALOG_URL` at your local clone and run the setup command against a test project:

   ```bash
   mkdir -p /tmp/test-project
   DEVCONTAINER_CATALOG_URL="/absolute/path/to/caylent-devcontainer-catalog" \
     cdevcontainer setup-devcontainer --catalog-entry <your-entry-name> /tmp/test-project
   ```

   Verify that the expected files are copied into `/tmp/test-project/.devcontainer/`.

4. **Verify the devcontainer builds:**

   Open the test project in VS Code and accept the "Reopen in Container" prompt, or build from the command line:

   ```bash
   cd /tmp/test-project
   devcontainer build --workspace-folder .
   ```

   Confirm that:
   - The container builds without errors
   - The `postCreateCommand` runs successfully
   - Tools and extensions specified in `devcontainer.json` are available
   - The development environment is functional for the intended workflow

### Verifying the DevContainer Build

When testing, check the following:

- Container starts without errors
- The postcreate script completes (check `/tmp/devcontainer-setup.log` inside the container)
- Language runtimes and tools are installed at expected versions
- VS Code extensions load correctly
- Port forwarding works for any specified ports
- Project-specific tooling (linters, formatters, build tools) is functional

---

## Modifying Common Assets

Files in `common/devcontainer-assets/` are shared across **all** entries in this catalog. Changes to these files affect every entry and every project that uses this catalog.

The three common asset files are:

| File | Purpose |
|------|---------|
| `.devcontainer.postcreate.sh` | Main setup script executed during container creation. Handles environment setup, tool installation, AWS configuration, git authentication, and more. |
| `devcontainer-functions.sh` | Shared bash functions (logging, WSL compatibility, proxy validation, git configuration) used by the postcreate script and project-setup.sh. |
| `project-setup.sh` | Template for project-specific setup. Developers customize this file after initial setup to add project-level initialization commands. |

**Guidelines for common asset changes:**

1. **Test against at least one entry** before opening a PR. Set up a test project using an existing entry and verify the postcreate script runs correctly with your changes.

2. **Document changes in the PR description.** Explain what changed, why, and what the impact is on existing entries and projects.

3. **Consider backward compatibility.** Existing projects that already use this catalog will pick up common asset changes on their next `setup-devcontainer` run. Avoid breaking changes to function signatures, expected environment variables, or script behavior without a migration path.

4. **Do not add entry-specific logic to common assets.** Common assets must remain general-purpose. If logic applies to only one entry, it belongs in that entry's files or in `project-setup.sh` customization.

---

## Pull Request Process

### Branch and Merge Strategy

| Item | Requirement |
|------|-------------|
| **Base branch** | `main` |
| **Branch naming** | Feature branch from `main` (e.g., `add-cna-java-spring`, `fix-postcreate-logging`) |
| **Required approvals** | 1 approving review |
| **Merge method** | Squash merge only |

### PR Requirements

Every pull request must include:

1. **Description** -- What changed and why. For new entries, describe what practice or team this serves and what tools are included.

2. **Impact assessment** (for common asset changes) -- Describe how the change affects existing entries. List which entries were tested.

3. **Validation confirmation** -- State that `cdevcontainer catalog validate --local .` passes. Include the output if helpful.

### Review Checklist

Reviewers and contributors should verify every item before approving or merging a pull request.

**Validation:**

- [ ] `cdevcontainer catalog validate --local .` passes with no errors

**Entry structure (for new or modified entries):**

- [ ] Entry directory name follows the naming pattern (`^[a-z][a-z0-9-]*[a-z0-9]$`, minimum 2 characters)
- [ ] `catalog-entry.json` exists with required `name` and `description` fields
- [ ] `name` in `catalog-entry.json` matches the directory name exactly
- [ ] `description` is meaningful and clearly describes the entry's purpose
- [ ] `tags` (if present) follow the naming pattern and are relevant to the entry
- [ ] `devcontainer.json` exists and contains a valid `postCreateCommand` referencing `.devcontainer/.devcontainer.postcreate.sh`
- [ ] `VERSION` file exists and contains a valid semver string
- [ ] No files conflict with common asset names (`.devcontainer.postcreate.sh`, `devcontainer-functions.sh`, `project-setup.sh`)
- [ ] No duplicate entry name in the catalog

**Quality:**

- [ ] `devcontainer.json` uses pinned or well-defined versions for features where appropriate
- [ ] Extensions listed in `devcontainer.json` are relevant to the entry's technology stack
- [ ] Entry does not include credentials, secrets, or environment-specific values
- [ ] VERSION has been incremented if this is a change to an existing entry

**Common asset changes (if applicable):**

- [ ] Change has been tested against at least one existing entry
- [ ] PR description includes an impact assessment
- [ ] Backward compatibility has been considered and documented
- [ ] No entry-specific logic has been added to common assets

**PR hygiene:**

- [ ] PR description explains the purpose and scope of changes
- [ ] Squash merge is selected (not merge commit or rebase)
- [ ] Only files relevant to the change are included
