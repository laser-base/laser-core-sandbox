# CI/CD Recommendations

This document describes the recommended GitHub Actions workflow setup for this repository, aligned with laser-base org standards.

## TL;DR

- **Two workflows** handle versioning and release: `bump-version.yml` and `release-publish.yml`
- **Bump** is manually triggered, requires approval, and uses a GitHub App to push — this is what chains into the release
- **Release** fires automatically on a `v*` tag push: builds, publishes to PyPI via OIDC, signs with Sigstore, creates a GitHub Release
- **The build job is repo-specific** — compiled packages need `cibuildwheel`, pure Python packages can use `uv build`; the publish, signing, and release jobs are standardized across all repos

## Workflows

### `bump-version.yml`

Manually triggered via the GitHub Actions UI. Prompts for `patch`, `minor`, or `major`.

Requires approval via the `restricted` environment before running. Uses a GitHub App (not `GITHUB_TOKEN`) to push the version commit and tag — this is what allows the tag push to trigger `release-publish.yml`. Runs `bump-my-version`, commits, tags, and pushes.

**Org-level secrets required:**

- `BOT_APP_ID` — GitHub App numeric ID
- `BOT_APP_PRIVATE_KEY` — GitHub App private key

### `release-publish.yml`

Triggered automatically when a `v*` tag is pushed (i.e. after every successful bump). Also has a manual `workflow_dispatch` trigger as an escape hatch.

**The build job is repo-specific.** Compiled packages should use `cibuildwheel` across an appropriate OS and Python version matrix. Pure Python packages can use `uv build` on a single Ubuntu runner. Either way, the build job must upload artifacts under the name `python-package-distributions` (or a pattern like `python-package-distributions-*` for multi-OS builds) — that's the handoff point to the standardized downstream jobs.

The following jobs are identical across all repos:
- Publishes to PyPI using OIDC trusted publishing — no API token stored in the repo
- Signs artifacts with Sigstore
- Creates a GitHub Release — only if PyPI publish succeeded

**PyPI trusted publishing setup (one time):**

1. Go to the `<PACKAGE_NAME>` package on PyPI → Manage → Publishing
2. Add a trusted publisher:
    - Owner: `laser-base`
    - Repository: `<REPO_NAME>`
    - Workflow: `release-publish.yml`
    - Environment: `restricted`

## Authorization

The `restricted` environment gate on the publish job can be removed once `v*` tag protection is enabled at the org level with the GitHub App as the sole bypass. At that point, the bump approval becomes the single authorization point — no human can push a release tag directly, so no second gate is needed on publish.

## End-to-end flow

```
Maintainer triggers bump-version.yml (patch/minor/major)
    → restricted environment approval required
    → bump-my-version updates pyproject.toml, commits, tags
    → GitHub App pushes commit + tag
        → tag push triggers release-publish.yml
            → build: repo-specific (cibuildwheel or uv build) → artifacts uploaded
            → publish-to-pypi: OIDC auth → published to PyPI
            → github-release: Sigstore sign → GitHub Release created
```
