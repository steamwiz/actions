# steamwiz/actions

Shared GitHub Actions reusable-workflow library for the Steamwiz GitHub organisation.
Any Steamwiz project (or any public GitHub user) can call these workflows with a single `uses:` line.

## Workflows

| Workflow | Purpose |
|---|---|
| [`python-pipeline.yml`](#python-pipelineyml) | Full CI/CD pipeline for a Python PyPI package |
| [`mdbook-build.yml`](#mdbook-buildyml) | Build an mdBook documentation site |
| [`cloudflare-deploy.yml`](#cloudflare-deployyml) | Deploy a static site to Cloudflare Pages |

---

## `python-pipeline.yml`

Full CI/CD pipeline: install → test → coverage → style → release → build.

PyPI publish is **not** included; see [PyPI Publish Pattern](#pypi-publish-pattern).

### Inputs

| Input | Type | Default | Description |
|---|---|---|---|
| `python-version` | string | `'3.13'` | Python version for `actions/setup-python` |
| `source-dir` | string | `''` | Source package directory. Defaults to repo name with hyphens → underscores. |
| `coverage-min` | number | `80` | Minimum coverage %; fails if below |
| `style-dirs` | string | `'test'` | Space-separated extra dirs for pycodestyle (beyond `source-dir`) |
| `do-pytest` | boolean | `true` | `false` to skip pytest and coverage |
| `do-coverage` | boolean | `true` | `false` to skip coverage check (pytest still runs) |
| `do-style` | boolean | `true` | `false` to skip pycodestyle |
| `do-release` | boolean | `true` | `false` to skip the release job |
| `release-level` | string | `''` | `patch`, `minor`, or `major` to force a release at that level |

### Secrets

| Secret | Required | Description |
|---|---|---|
| `GH_TOKEN` | Yes (release only) | GitHub token with `contents: write`, `issues: write` for python-semantic-release |

Pass via `secrets: inherit` from the caller.

### Outputs

| Output | Description |
|---|---|
| `version` | Released version string (e.g. `1.4.2`), or empty |
| `released` | `'true'` if a release occurred, `'false'` otherwise |
| `tag` | Git tag created (e.g. `v1.4.2`), or empty |

### Minimal caller example

```yaml
# .github/workflows/pipeline.yml
name: Pipeline

on:
  push:
    branches: [main]
  pull_request:

jobs:
  python:
    uses: steamwiz/actions/.github/workflows/python-pipeline.yml@v1
    secrets: inherit
```

### Full caller example (with publish and docs)

```yaml
# .github/workflows/pipeline.yml
name: Pipeline

on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:
    inputs:
      release-level:
        description: "Force release level (patch/minor/major/none)"
        required: false
        default: none

jobs:
  python:
    uses: steamwiz/actions/.github/workflows/python-pipeline.yml@v1
    with:
      python-version: '3.13'
      coverage-min: 96
      do-release: ${{ github.event_name != 'workflow_dispatch' || inputs.release-level != 'none' }}
      release-level: ${{ github.event_name == 'workflow_dispatch' && inputs.release-level != 'none' && inputs.release-level || '' }}
    secrets: inherit

  publish:
    needs: python
    if: needs.python.outputs.released == 'true'
    uses: ./.github/workflows/publish.yml
    secrets: inherit

  build-docs:
    uses: steamwiz/actions/.github/workflows/mdbook-build.yml@v1

  deploy-docs:
    needs: [python, build-docs]
    if: github.ref == 'refs/heads/main'
    uses: steamwiz/actions/.github/workflows/cloudflare-deploy.yml@v1
    with:
      cloudflare-project-name: my-project
      directory: html
    secrets: inherit
```

### `pyproject.toml` configuration required

Projects must include this block (or equivalent):

```toml
[tool.semantic_release]
tag_format = "v{version}"
version_toml = ["pyproject.toml:tool.poetry.version"]
build_command = false

[tool.semantic_release.branches.main]
match = "main"
prerelease = false

[tool.semantic_release.remote.token]
env = "GH_TOKEN"

[tool.semantic_release.publish]
upload_to_vcs_release = false
```

Start with `version = "0.0.0"` in `[tool.poetry]`; python-semantic-release will manage it.

---

## `mdbook-build.yml`

Builds an mdBook site and uploads it as an artifact.

### Inputs

| Input | Type | Default | Description |
|---|---|---|---|
| `mdbook-version` | string | `'latest'` | mdBook version to install |
| `output-dir` | string | `'html'` | Build output directory (must match `book.toml`) |
| `artifact-name` | string | `'mdbook-site'` | Name of the uploaded artifact |
| `artifact-retention-days` | number | `7` | Artifact retention in days |

### Outputs

| Output | Description |
|---|---|
| `artifact-name` | The artifact name (for downstream reference) |

### Example

```yaml
build-docs:
  uses: steamwiz/actions/.github/workflows/mdbook-build.yml@v1
```

---

## `cloudflare-deploy.yml`

Downloads a static site artifact and deploys it to Cloudflare Pages using
`cloudflare/wrangler-action`.

### Inputs

| Input | Type | Default | Description |
|---|---|---|---|
| `cloudflare-project-name` | string | — **(required)** | Cloudflare Pages project name |
| `artifact-name` | string | `'mdbook-site'` | Artifact to download and deploy |
| `directory` | string | `'html'` | Subdirectory within the artifact to deploy |

### Secrets

| Secret | Required | Description |
|---|---|---|
| `CLOUDFLARE_ACCOUNT_ID` | Yes | Cloudflare account ID (32 hex chars) |
| `CLOUDFLARE_API_TOKEN` | Yes | API token with Pages edit permission |

### Example

```yaml
deploy-docs:
  needs: build-docs
  if: github.ref == 'refs/heads/main'
  uses: steamwiz/actions/.github/workflows/cloudflare-deploy.yml@v1
  with:
    cloudflare-project-name: my-project
    directory: html
  secrets: inherit
```

---

## PyPI Publish Pattern

PyPI's OIDC trusted publisher issues tokens per-job. Because the publish step must
live in the **project's own repository** (not in `steamwiz/actions`), the shared
library does not include a publish workflow.

Each project creates `.github/workflows/publish.yml`:

```yaml
name: Publish to PyPI

on:
  workflow_call:

jobs:
  pypi-publish:
    runs-on: ubuntu-latest
    environment: pypi
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: dist
          path: dist/
      - uses: pypa/gh-action-pypi-publish@release/v1
```

Register a GitHub Actions trusted publisher on PyPI with:

| Field | Value |
|---|---|
| Owner | `steamwiz` |
| Repository | your project repo (e.g. `busy`) |
| Workflow filename | `publish.yml` |
| Environment | `pypi` |

Create a GitHub Environment named `pypi` in the project repo settings.

---

## Versioning

| Ref | Meaning |
|---|---|
| `@v1` | Floating major-version tag; always the latest `v1.x.y` |
| `@v1.2.3` | Pinned exact release |
| `@main` | HEAD; for testing only |

Pin callers to `@v1` for automatic non-breaking updates. Major version increments
only on breaking changes to workflow inputs or job structure.

## Action Pinning

All third-party actions are pinned to full-length commit SHAs. Dependabot keeps
these current via weekly PRs. See `SPEC.md §12` for the full pinning table.
