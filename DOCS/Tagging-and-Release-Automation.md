
#  Tagging and Release Automation with GitHub Actions

This documentation outlines the complete setup for automated **Semantic Versioning**, **Commit Linting**, **Tagging**, **CHANGELOG updates**, and **GitHub Releases** using GitHub Actions.

---

##  Semantic Versioning Rules

We follow **Semantic Versioning (SemVer)** in the format: `MAJOR.MINOR.PATCH`

| Commit Type        | Current Version | New Version |
|--------------------|-----------------|-------------|
| `feat:`            | `v1.2.3`        | `v1.3.0`    |
| `fix:`             | `v1.2.3`        | `v1.2.4`    |
| `BREAKING CHANGE:` | `v1.2.3`        | `v2.0.0`    |

- **MAJOR**: Incompatible API changes (breaking changes)
- **MINOR**: New features (non-breaking)
- **PATCH**: Bug fixes

---

##  GitHub Actions Workflow Setup

### Reusable Workflow File

**Path:** `Mahesh2511/AUTO-Release-Tagging/.github/workflows/TaggingReleaseAutomation.yaml`

This reusable workflow performs:
- Commit message linting (using `commitlint`)
- Semantic version tag generation
- Git tag creation and push
- Release notes generation
- CHANGELOG.md update and commit
- GitHub Release creation

---

### Caller Workflow Example

To use this reusable workflow in your repository, create a caller workflow like this:

```yaml
name: Call Reusable Tagging Workflow

permissions:
  contents:write

on:
  workflow_dispatch:
  push:
    branches:
      - release  # Branch to trigger tagging

jobs:
  trigger_tag_release:
    uses: Mahesh2511/AUTO-Release-Tagging/.github/workflows/TaggingReleaseAutomation.yaml@main
    with:
      branch: release
```

**Explanation:**

- The `uses:` field references the reusable workflow's repo, path, and branch.
- The `branch` input specifies which branch triggers the release automation.
- This caller workflow triggers on push events to the `release` branch or manually via `workflow_dispatch`.

---

### Changing Branch for Tagging & Release

To enable tagging and release automation on a different branch (for example, `main` or `develop`), update:

1. The `branches` section under the `push` event in the caller workflow:

```yaml
on:
  workflow_dispatch:
  push:
    branches:
      - main  # Change to your desired branch
```

2. The `branch` input under `with`:

```yaml
with:
  branch: main  # Match this to the branch above
```

This ensures your reusable workflow runs when pushing to your chosen branch and uses the correct branch context internally.

---

## Commit Linting Configuration

Create a `commitlint.config.js` file in your repository root with the following content:

```js
module.exports = {
  extends: ["@commitlint/config-conventional"],
  rules: {
    "subject-case": [0, "never"],       // Ignore case rules on subject line
    "subject-empty": [0, "never"],      // Allow empty subjects (no warnings)
    "type-empty": [2, "never"],         // Disallow empty type in commit
    "type-enum": [2, "always", ["feat", "fix", "BREAKING CHANGE"]],  // Allowed types
  },
};
```

### Explanation:

- Uses conventional commit types: `feat`, `fix`, and `BREAKING CHANGE`.
- Ignores subject case and empty subject warnings for flexibility.
- Enforces that commit types must not be empty and must be one of the allowed types.

---

## Commit Message Examples

- `feat: add user profile page`
- `fix: resolve logout bug`
- `BREAKING CHANGE: upgrade API version`

These commit messages influence the version bump:

- `feat` → Minor version bump
- `fix` → Patch version bump
- `BREAKING CHANGE` → Major version bump

---

## Release Notes Format

Generated release notes look like this:

```md
## Release v1.3.0

### New Features
- feat: add user profile page

### Bug Fixes
- fix: crash on empty response

### Breaking Changes
- BREAKING CHANGE: refactor database layer

### Known Issues
 Please manually add known issues here before publishing the release.
```

---

## CHANGELOG.md Format

Each release appends to `CHANGELOG.md` in this format:

```md
### v1.3.0 - 2025-05-23
## Release v1.3.0

### New Features
- feat: add user profile page

### Bug Fixes
- fix: crash on empty response

### Breaking Changes
- BREAKING CHANGE: refactor database layer

### Known Issues
 Please manually add known issues here before publishing the release.
```

---

## Setup Requirements

- `commitlint.config.js` present in the root directory
- Valid commit messages following configured rules
- GitHub Actions workflows configured with appropriate permissions

---

## Optional: Local Commit Linting Setup

To lint commits locally before pushing:

```bash
npm install --save-dev husky @commitlint/config-conventional @commitlint/cli
npx husky install
npx husky add .husky/commit-msg 'npx --no -- commitlint --edit "$1"'
```

---

## Contribution Guidelines

- Write valid semantic commit messages.
- Ensure commitlint passes locally or in the CI.
- Push code to the designated branch (e.g., `release`) to trigger automated versioning.

---

## FAQ

### Why are my commits failing the linting step?

Make sure commit messages follow the pattern:

- `feat: description`
- `fix: description`
- `BREAKING CHANGE: description`

### How do I trigger the versioning workflow manually?

Use the GitHub UI to trigger `workflow_dispatch` on the caller workflow.

### Can I run this workflow on branches other than `release`?

Yes, update the caller workflow’s `push` branches and `branch` input accordingly.

### What if I want to edit release notes or changelog manually?

You can edit release notes before publishing or modify the changelog anytime since they are committed back to the repo.

---

## Contact

For help or questions, contact the DevOps Team or open an issue in the repo.
Contact the : `pawarmahesh2511@gmail.com`

---
