#  Versioning and Release Automation - Workflow Explanation

This GitHub Actions workflow automates semantic versioning, changelog updates, release note generation, and GitHub Releases creation based on commit messages.

##  Trigger: `workflow_call`
This workflow is designed to be **reused** as a callable workflow.

### Input
- `branch`: The branch name to perform the release from (e.g., `main`, `release/x.y`, etc.)

---

##  Job: `tag_release`
Executes the full release pipeline on an Ubuntu runner.

###  Step 1: Checkout Repository

```yaml
- name: Checkout Repository
  uses: actions/checkout@v4
  with:
    fetch-depth: 0  # Full history and tags
```

Ensures the full commit history and all tags are available for accurate semantic version calculation.

---

###  Step 2: Install Commitlint Dependencies

```yaml
- name: Install Commitlint Dependencies
  run: |
    npm install --save-dev @commitlint/config-conventional @commitlint/cli
```

Prepares tools to enforce conventional commit message standards.

---

###  Step 3: Lint Commit Messages

```yaml
- name: Lint Commit Messages
  run: |
    npx commitlint --from=HEAD~1 --to=HEAD
```

Validates the format of the most recent commit. Ensures it follows the conventional commit syntax (`feat:`, `fix:`, `BREAKING CHANGE`, etc.).

---

###  Step 4: Fetch All Tags

```yaml
- name: Fetch All Tags
  run: git fetch --tags
```

Ensures local Git context includes all tags for version comparison.

---

###  Step 5: Generate Semantic Version Tag

```yaml
- name: Generate Semantic Version Tag
  id: generate_tag
  run: |
    latest_tag=$(git tag --sort=-creatordate --merged HEAD | grep -E "^v[0-9]+\.[0-9]+\.[0-9]+$" | head -n 1)
    echo "LATEST_TAG=$latest_tag" >> $GITHUB_ENV

    if [ -z "$latest_tag" ]; then
      new_tag="v1.0.0"
    else
      major=$(echo $latest_tag | cut -d '.' -f 1 | sed 's/v//')
      minor=$(echo $latest_tag | cut -d '.' -f 2)
      patch=$(echo $latest_tag | cut -d '.' -f 3)

      commit_messages=$(git log $latest_tag..HEAD --pretty=%B)

      if echo "$commit_messages" | grep -q "BREAKING CHANGE"; then
        new_tag="v$((major + 1)).0.0"
      elif echo "$commit_messages" | grep -q "^feat:"; then
        new_tag="v$major.$((minor + 1)).0"
      else
        new_tag="v$major.$minor.$((patch + 1))"
      fi
    fi

    echo "NEW_TAG=$new_tag" >> $GITHUB_ENV
    echo "Generated new tag: $new_tag"
```

This dynamically determines the **next semantic version tag** based on:
- `BREAKING CHANGE` → major bump
- `feat:` → minor bump
- others → patch bump

---

###  Step 6: Run Tests

```yaml
- name: Run Tests
  run: echo "Running tests for release..."
```

Placeholder for test automation. You can replace this with real tests (e.g., `npm test`, `pytest`, etc.).

---

###  Step 7: Create and Push Git Tag

```yaml
- name: Create and Push Tag
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  run: |
    git config --global user.name "github-actions"
    git config --global user.email "github-actions@github.com"

    git tag ${{ env.NEW_TAG }}
    git push https://x-access-token:${GITHUB_TOKEN}@github.com/${{ github.repository }}.git ${{ env.NEW_TAG }}
```

Creates a Git tag locally and pushes it to the remote repository.

---

###  Step 8: Generate Release Notes

```yaml
- name: Generate Release Notes
  id: release_notes
  run: |
    echo "## Release ${{ env.NEW_TAG }}" > release_notes.md

    echo "### New Features" >> release_notes.md
    feat_commits=$(git log ${{ env.LATEST_TAG }}..HEAD --pretty=format:"- %s" --grep="^feat:")
    [ -z "$feat_commits" ] && echo "No new features in this release." >> release_notes.md || echo "$feat_commits" >> release_notes.md

    echo "### Bug Fixes" >> release_notes.md
    fix_commits=$(git log ${{ env.LATEST_TAG }}..HEAD --pretty=format:"- %s" --grep="^fix:")
    [ -z "$fix_commits" ] && echo "No bug fixes in this release." >> release_notes.md || echo "$fix_commits" >> release_notes.md

    echo "### Breaking Changes" >> release_notes.md
    breaking_commits=$(git log ${{ env.LATEST_TAG }}..HEAD --pretty=format:"- %s" --grep="BREAKING CHANGE")
    [ -z "$breaking_commits" ] && echo "No breaking changes in this release." >> release_notes.md || echo "$breaking_commits" >> release_notes.md

    echo "### Known Issues" >> release_notes.md
    echo "⚠️ Please manually add known issues here before publishing the release." >> release_notes.md
```

Parses commit history and creates structured release notes:
- New Features (from `feat:`)
- Bug Fixes (from `fix:`)
- Breaking Changes
- Manual placeholder for Known Issues

---

###  Step 9: Update `CHANGELOG.md`

```yaml
- name: Update CHANGELOG.md
  run: |
    echo "### ${{ env.NEW_TAG }} - $(date +'%Y-%m-%d')" >> CHANGELOG.md
    cat release_notes.md >> CHANGELOG.md
    echo "" >> CHANGELOG.md
    git add CHANGELOG.md
    git commit -m "chore: update CHANGELOG for ${{ env.NEW_TAG }}"
    git push origin HEAD
```

Appends the new release notes to `CHANGELOG.md` and commits the update.

---

###  Step 10: Create GitHub Release

```yaml
- name: Create GitHub Release Using `actions/github-script`
  uses: actions/github-script@v6
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    NEW_TAG: ${{ env.NEW_TAG }}
  with:
    script: |
      const fs = require('fs');
      const releaseNotes = fs.readFileSync('release_notes.md', 'utf8');
      const newTag = process.env.NEW_TAG;

      await github.rest.repos.createRelease({
        owner: context.repo.owner,
        repo: context.repo.repo,
        tag_name: newTag,
        name: `Release ${newTag}`,
        body: releaseNotes.trim(),
        draft: false,
        prerelease: false,
        generate_release_notes: true
      });

      console.log(`Release created with tag ${newTag}`);
```

Uses the `actions/github-script` to publish a new GitHub release with custom notes.

---

##  Commit Message Impact on Version

If the commit linter is ignored:
- **Any commit** that does not match `feat:` or `BREAKING CHANGE` → treated as **patch** by default.

You can control the version bump by structuring commit messages properly.

---

##  Customizing for Different Branches

In the caller workflow, you can specify the branch to release from:

```yaml
uses: ./.github/workflows/release.yaml
with:
  branch: 'release/1.2'
```

The reusable workflow will operate based on this input, and tag creation will be done from that branch's history.

---

##  Summary

| Step | Purpose |
|------|---------|
| Checkout | Get full code and tags |
| Commit Lint | Ensure conventional commits |
| Tag Gen | Auto version tag generation |
| Push Tag | Create and push new version |
| Release Notes | Generate from commit logs |
| Changelog | Append to project changelog |
| GitHub Release | Publish official release |


