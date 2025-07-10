# Reusable GitHub Workflows

A collection of reusable GitHub Actions workflows designed to streamline development processes, enforce code standards, and automate common CI/CD tasks. These workflows can be easily integrated into any GitHub repository to improve development workflow consistency and automation.

## Table of Contents

- [Available Workflows](#available-workflows)
  - [Pull Request Validation](#pull-request-validation)
    - [PR Title Check](#pr-title-check-pr-title-checkyml)
    - [PR Branch Name Check](#pr-branch-name-check-pr-branch-name-checkyml)
  - [Container Operations](#container-operations)
    - [Container Build and Push](#container-build-and-push-container-build-pushyml)
    - [Container Check Existing Image](#container-check-existing-image-container-check-existing-imageyml)
    - [Container Retag and Push](#container-retag-and-push-container-retag-pushyml)
  - [Utility Workflows](#utility-workflows)
    - [Branch Latest Non-Merge Commit SHA](#branch-latest-non-merge-commit-sha-branch-latest-nonmerge-commit-shayml)
    - [Update Major Branch](#update-major-branch-_update-major-branchyml)
- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Setup Instructions](#setup-instructions)
  - [Example: Complete PR Validation Setup](#example-complete-pr-validation-setup)
  - [Example: Container CI/CD Pipeline](#example-container-cicd-pipeline)
- [Workflow Requirements](#workflow-requirements)
  - [Common Requirements](#common-requirements)
  - [JIRA Integration](#jira-integration)
- [Customization](#customization)
  - [Modifying Validation Rules](#modifying-validation-rules)
  - [Adding New Commit Types](#adding-new-commit-types)
- [Related Resources](#related-resources)

## Available Workflows

### Pull Request Validation

#### PR Title Check (`pr-title-check.yml`)
Validates pull request titles against Conventional Commits format with JIRA ticket integration.

**Features:**
- Enforces structured PR titles: `<type>(<JIRA-ticket>): <description>`
- Supports all standard commit types (feat, fix, chore, docs, test, refactor, perf, ci, build, style, revert)
- Automatically excludes Dependabot PRs
- Enables automated changelog generation and semantic versioning

**Usage:**
```yaml
jobs:
  validate-title:
    uses: ./.github/workflows/pr-title-check.yml
    with:
      pr_title: ${{ github.event.pull_request.title }}
      branch_name: ${{ github.head_ref }}
```

**Examples of valid titles:**
- `feat(PROJECT-123): Add OAuth2 authentication`
- `fix(BUG-456): Resolve memory leak in data processing`
- `docs(DOC-123): Update API documentation`

#### PR Branch Name Check (`pr-branch-name-check.yml`)
Validates branch names follow consistent naming conventions with JIRA integration.

**Features:**
- Enforces branch naming: `<type>/<JIRA-ticket>-<description>`
- Improves traceability between branches and tickets
- Automatically excludes Dependabot branches

**Usage:**
```yaml
jobs:
  validate-branch:
    uses: ./.github/workflows/pr-branch-name-check.yml
    with:
      branch_name: ${{ github.head_ref }}
```

**Examples of valid branch names:**
- `feat/PROJECT-123-add-user-authentication`
- `fix/BUG-456-memory-leak-fix`
- `docs/DOC-123-api-documentation-update`

### Container Operations

#### Container Build and Push (`container-build-push.yml`)
Builds and pushes container images to registries using Podman.

**Features:**
- Support for multiple container registries (GitHub Container Registry, Docker Hub, Azure ACR, etc.)
- Flexible container file support (Dockerfile, Containerfile)
- Automatic tagging with commit SHA and latest
- Built-in authentication handling

**Usage:**
```yaml
jobs:
  build-and-push:
    uses: ./.github/workflows/container-build-push.yml
    with:
      image_name: "myorg/myapp"
      image_registry: "ghcr.io"
      build_context: "./src"
      container_file_name: "Dockerfile"
      commit_sha: ${{ github.sha }}
    secrets:
      image_registry_password: ${{ secrets.GITHUB_TOKEN }}
```

#### Container Check Existing Image (`container-check-existing-image.yml`)
Checks if a container image already exists in the registry before building.

**Features:**
- Prevents unnecessary rebuilds
- Supports registry authentication
- Returns existence status for conditional workflows

#### Container Retag and Push (`container-retag-push.yml`)
Retags existing container images with new tags and pushes them to registries.

**Features:**
- Efficient image retagging without rebuilding
- Support for multiple target registries
- Useful for promoting images between environments

### Utility Workflows

#### Branch Latest Non-Merge Commit SHA (`branch-latest-nonmerge-commit-sha.yml`)
Retrieves the SHA of the latest non-merge commit from the current branch.

**Features:**
- Excludes merge commits to get actual code changes
- Useful for deployment tracking and release tagging
- Falls back gracefully if no non-merge commits exist

**Usage:**
```yaml
jobs:
  get-commit:
    uses: ./.github/workflows/branch-latest-nonmerge-commit-sha.yml
  
  deploy:
    needs: get-commit
    runs-on: ubuntu-latest
    steps:
      - name: Deploy using commit SHA
        run: echo "Deploying ${{ needs.get-commit.outputs.commit_sha }}"
```

#### Update Major Branch (`_update-major-branch.yml`)
Internal workflow for maintaining major version branches.

## Getting Started

### Prerequisites
- GitHub repository with Actions enabled
- Appropriate secrets configured for container registries (if using container workflows)
- JIRA project integration (for PR validation workflows)

### Setup Instructions

1. **Copy workflows**: Copy the desired workflow files to your repository's `.github/workflows/` directory.

2. **Configure secrets**: Set up necessary secrets in your repository settings:
   - `GITHUB_TOKEN` (automatically available)
   - Container registry credentials (if using container workflows)

3. **Create workflow files**: Create calling workflows in your repository that reference these reusable workflows.

### Example: Complete PR Validation Setup

Create `.github/workflows/pr-validation.yml` in your repository:

```yaml
name: PR Validation

on:
  pull_request:
    types: [opened, synchronize, reopened, edited]

jobs:
  validate-title:
    uses: ./.github/workflows/pr-title-check.yml
    with:
      pr_title: ${{ github.event.pull_request.title }}
      branch_name: ${{ github.head_ref }}

  validate-branch:
    uses: ./.github/workflows/pr-branch-name-check.yml
    with:
      branch_name: ${{ github.head_ref }}
```

### Example: Container CI/CD Pipeline

Create `.github/workflows/container-ci.yml` in your repository:

```yaml
name: Container CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  check-existing:
    uses: ./.github/workflows/container-check-existing-image.yml
    with:
      image_name: "myorg/myapp"
      image_registry: "ghcr.io"
      image_tag: ${{ github.sha }}
    secrets:
      image_registry_password: ${{ secrets.GITHUB_TOKEN }}

  build-and-push:
    needs: check-existing
    if: needs.check-existing.outputs.image_exists == 'false'
    uses: ./.github/workflows/container-build-push.yml
    with:
      image_name: "myorg/myapp"
      image_registry: "ghcr.io"
      commit_sha: ${{ github.sha }}
    secrets:
      image_registry_password: ${{ secrets.GITHUB_TOKEN }}
```

## Workflow Requirements

### Common Requirements
- **Permissions**: Most workflows require `contents: read` permission minimum
- **Container workflows**: Require `packages: write` for registry operations
- **Secrets**: Container workflows need registry authentication secrets

### JIRA Integration
For PR validation workflows, ensure your JIRA tickets follow the format:
- Uppercase project code + hyphen + numbers
- Examples: `PROJECT-123`, `BUG-456`, `FEAT-789`

## Customization

### Modifying Validation Rules
You can customize the validation patterns by editing the regex patterns in the workflow files:

- **PR Title Pattern**: Modify `JIRA_PATTERN` in `pr-title-check.yml`
- **Branch Name Pattern**: Modify `BRANCH_PATTERN` in `pr-branch-name-check.yml`
- **Supported Types**: Update the type lists in both validation workflows

### Adding New Commit Types
To add new commit types (e.g., `security`, `hotfix`):

1. Update the regex pattern in the validation workflows
2. Add the new type to the documentation comments
3. Test with example PR titles/branch names

## Related Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Reusable Workflows Guide](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
- [Conventional Commits Specification](https://www.conventionalcommits.org/)
- [Semantic Versioning](https://semver.org/)

---

*These workflows are designed to be flexible and customizable. Adapt them to fit your team's specific development standards and requirements.*
