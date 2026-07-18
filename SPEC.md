# Specification: steamwiz/actions

> Repository: `github.com/steamwiz/actions`  
> Status: Design — pre-implementation  
> Initial consumer: `steamwiz/busy`  
> Scope: Family of Python PyPI projects

---

## 1. Purpose

`steamwiz/actions` is a shared GitHub Actions reusable-workflow library for the
Steamwiz GitHub organisation. It provides standardised, versioned CI/CD pipelines
that any Steamwiz project can call with a single `uses:` line.

---

## 2. Repository Layout

```
actions/
├── .github/
│   └── workflows/
│       ├── python-pipeline.yml        # Full build/test/release pipeline (workflow_call)
│       ├── mdbook-build.yml           # Build mdBook documentation (workflow_call)
│       └── cloudflare-deploy.yml      # Deploy static site to Cloudflare Pages (workflow_call)
├── README.md
└── CHANGELOG.md
```

GitHub requires all reusable workflows to live directly under `.github/workflows/`.
Subdirectories are not supported for `workflow_call` targets.

The repository is **public**, so any Steamwiz project (and any other GitHub user)
can reference it without PAT configuration.

---

## 3. Versioning

The repository uses **semantic version tags** (`v1`, `v1.2`, `v1.2.3`) on the
`main` branch. Callers pin to a major version tag (e.g. `@v1`) which is a moving
tag that always points to the latest `v1.x.y` release. This mirrors the convention
used by first-party GitHub Actions.

- `@v1` — floating major-version tag (updated on each `v1.x.y` release)  
- `@v1.2.3` — pinned exact release  
- `@main` — HEAD; use only for testing

Breaking changes in the reusable workflow inputs or job structure increment the
major version.

---

## 4. Reusable Workflow: `python-pipeline.yml`

### 4.1 Purpose

Full CI/CD pipeline for a Python package published to PyPI. Covers:

1. Install dependencies (Poetry)
2. Run tests (pytest + coverage)
3. Style check (pycodestyle)
4. Release: version bump, git tag, GitHub Release, changelog (python-semantic-release)
5. Build distribution (poetry build)

**PyPI publish is explicitly excluded.** See §7 for the rationale and the
project-side publish pattern.

### 4.2 Trigger

```yaml
on:
  workflow_call:
```

### 4.3 Inputs

| Input | Type | Default | Description |
|---|---|---|---|
| `python-version` | string | `'3.13'` | Python version passed to `actions/setup-python` |
| `coverage-min` | number | `80` | Minimum coverage percentage; job fails if below this |
| `style-dirs` | string | `'test'` | Space-separated list of directories (beyond the source dir) to include in pycodestyle |
| `do-pytest` | boolean | `true` | Set to `false` to skip pytest and coverage jobs |
| `do-coverage` | boolean | `true` | Set to `false` to skip the coverage check (pytest still runs) |
| `do-style` | boolean | `true` | Set to `false` to skip pycodestyle |
| `do-release` | boolean | `true` | Set to `false` to skip the release job entirely (useful for PR pipelines or manual skips) |

### 4.4 Secrets

| Secret | Required | Description |
|---|---|---|
| `GH_TOKEN` | Yes (release job only) | GitHub token with `contents: write` and `issues: write` for python-semantic-release to push tags, create GitHub Releases, and update changelogs. Use `secrets: inherit` from the caller so the caller's `GITHUB_TOKEN` flows through, or pass an org-level PAT. |

### 4.5 Outputs

| Output | Source job | Description |
|---|---|---|
| `version` | `release` | The version string released (e.g. `1.4.2`), or empty if no release occurred |
| `released` | `release` | `'true'` if a new version was released, `'false'` otherwise |
| `tag` | `release` | The git tag created (e.g. `v1.4.2`), or empty if no release |

Outputs are wired through from the `release` job so that the caller's workflow can
use them — for example, to conditionally run a publish job.

### 4.6 Job Graph

```
install ──┬──► pytest ──► coverage
          │
          ├──► style
          │
          └──► release  (main branch only; conventional commit driven)
                  │
                  └──► build   (only when released == true)
```

All jobs run on `ubuntu-latest` unless otherwise noted.

### 4.7 Job Specifications

#### `install`

- **Runner**: `ubuntu-latest`
- **Steps**:
  1. `actions/checkout@v4` with `fetch-depth: 0` (needed for semantic release to read commit history)
  2. `actions/setup-python@v5` with the `python-version` input
  3. `actions/cache@v4` keyed on `${{ runner.os }}-poetry-${{ hashFiles('**/poetry.lock') }}`; cache path `.venv`
  4. `pip install poetry`
  5. `poetry install --no-root`
- **Outputs**: The `.venv` directory is uploaded as a workflow artifact named `venv` (archived). Subsequent jobs download it rather than re-installing.

> **Note on venv artifact**: GitHub Actions does not share the filesystem between
> jobs. The venv is uploaded with `actions/upload-artifact@v4` and each downstream
> job downloads it with `actions/download-artifact@v4` before running. Cache is
> also used so repeat runs on the same branch are fast without the artifact round-trip.

#### `pytest`

- **Needs**: `install`
- **Condition**: `inputs.do-pytest == true`
- **Steps**:
  1. `actions/checkout@v4`
  2. `actions/setup-python@v5`
  3. Download `venv` artifact
  4. `poetry run python -m coverage run --source=<source-dir> -m pytest test/ -s --junitxml=junit/test-results.xml`
- **Source dir detection**: The source directory defaults to the repository name with hyphens replaced by underscores (matching Python package convention). This is computed as `${{ github.event.repository.name }}` with hyphens converted to underscores. If a project deviates from this convention it can set an env var or the caller can override via a future `source-dir` input.
- **Artifacts uploaded**: `.coverage`, `junit/` (JUnit XML); retained 7 days
- **Test results**: Upload JUnit XML with `actions/upload-artifact@v4`; report in PR checks via `EnricoMi/publish-unit-test-result-action@v2` (summary only, not blocking)

#### `coverage`

- **Needs**: `pytest`
- **Condition**: `inputs.do-coverage == true`
- **Steps**:
  1. `actions/setup-python@v5`
  2. `pip install coverage`
  3. Download `.coverage` artifact
  4. `python -m coverage xml`
  5. `python -m coverage report -m --fail-under ${{ inputs.coverage-min }}`
- **Coverage reporting**: Upload `coverage.xml` as artifact. Add a step using `irongut/CodeCoverageSummary@v1.3.0` to post a coverage comment on PRs.

#### `style`

- **Needs**: `install`
- **Condition**: `inputs.do-style == true`
- **Steps**:
  1. `actions/checkout@v4`
  2. `actions/setup-python@v5`
  3. Download `venv` artifact
  4. `poetry run pycodestyle <source-dir>`
  5. For each directory in `inputs.style-dirs`: `poetry run pycodestyle <dir>` (skip if directory does not exist)

#### `release`

- **Needs**: `pytest`, `coverage`, `style` (all must pass)
- **Condition**: `github.ref == 'refs/heads/main'` AND `inputs.do-release == true`
- **Permissions required**: `contents: write`, `issues: write`, `pull-requests: write`
- **Steps**:
  1. `actions/checkout@v4` with `fetch-depth: 0` and `token: ${{ secrets.GH_TOKEN }}`
  2. `actions/setup-python@v5`
  3. `pip install python-semantic-release`
  4. Configure git user (name: `github-actions[bot]`, email: `41898282+github-actions[bot]@users.noreply.github.com`)
  5. `semantic-release version` — reads conventional commits, bumps version in `pyproject.toml`, commits, tags, pushes, creates GitHub Release with changelog
  6. Capture outputs: `version`, `released`, `tag` from semantic-release exit code and output
- **python-semantic-release configuration** (expected in caller's `pyproject.toml`):
  ```toml
  [tool.semantic_release]
  version_toml = ["pyproject.toml:tool.poetry.version"]
  branch = "main"
  upload_to_pypi = false          # PyPI publish is handled project-side
  upload_to_release = false       # No dist assets in the GitHub Release
  build_command = ""              # Build handled in separate job
  changelog_file = "CHANGELOG.md"
  commit_version_number = true
  tag_format = "v{version}"
  ```
  Projects must have this block (or equivalent) in their `pyproject.toml`. The
  spec will provide a canonical snippet in the README.

#### `build`

- **Needs**: `release`
- **Condition**: `needs.release.outputs.released == 'true'`
- **Steps**:
  1. `actions/checkout@v4` with `ref: ${{ needs.release.outputs.tag }}` — checks out the tagged commit so the version in `pyproject.toml` is correct
  2. `actions/setup-python@v5`
  3. Download `venv` artifact
  4. `poetry build`
  5. Upload `dist/` as artifact named `dist`, retained 30 days
- The `dist` artifact is what the caller's publish job will consume.

### 4.8 Example Caller Workflow

```yaml
# busy/.github/workflows/pipeline.yml
name: Pipeline

on:
  push:
    branches: [main]
  pull_request:

jobs:

  python:
    uses: steamwiz/actions/.github/workflows/python-pipeline.yml@v1
    with:
      python-version: '3.13'
      coverage-min: 96
      do-style: true
    secrets: inherit

  publish:
    needs: python
    if: needs.python.outputs.released == 'true'
    uses: ./.github/workflows/publish.yml   # project-side; see §7
    secrets: inherit

  build-docs:
    uses: steamwiz/actions/.github/workflows/mdbook-build.yml@v1

  deploy-docs:
    needs: [python, build-docs]
    if: github.ref == 'refs/heads/main'
    uses: steamwiz/actions/.github/workflows/cloudflare-deploy.yml@v1
    with:
      cloudflare-project-name: steamwiz-busy
    secrets: inherit
```

---

## 5. Reusable Workflow: `mdbook-build.yml`

### 5.1 Purpose

Build an mdBook documentation site and upload the result as a workflow artifact
for a subsequent deploy step to consume.

### 5.2 Trigger

```yaml
on:
  workflow_call:
```

### 5.3 Inputs

| Input | Type | Default | Description |
|---|---|---|---|
| `mdbook-version` | string | `'latest'` | Version of mdBook to install via `peaceiris/actions-mdbook` |
| `output-dir` | string | `'html'` | Directory mdBook builds into (matches `book.toml` `build.build-dir`) |
| `artifact-name` | string | `'mdbook-site'` | Name of the uploaded artifact |
| `artifact-retention-days` | number | `7` | Artifact retention in days |

### 5.4 Secrets

None.

### 5.5 Outputs

| Output | Description |
|---|---|
| `artifact-name` | The artifact name (echoes the input, for downstream reference) |

### 5.6 Job Specification

#### `build`

- **Runner**: `ubuntu-latest`
- **Condition**: Runs unconditionally (callers gate on branch if needed)
- **Steps**:
  1. `actions/checkout@v4`
  2. `peaceiris/actions-mdbook@v2` with `mdbook-version: ${{ inputs.mdbook-version }}`
  3. `mdbook build --dest-dir ${{ inputs.output-dir }}`
  4. `actions/upload-artifact@v4` — artifact named `${{ inputs.artifact-name }}`; path `${{ inputs.output-dir }}/`

---

## 6. Reusable Workflow: `cloudflare-deploy.yml`

### 6.1 Purpose

Deploy a static site artifact to Cloudflare Pages.

### 6.2 Trigger

```yaml
on:
  workflow_call:
```

### 6.3 Inputs

| Input | Type | Default | Description |
|---|---|---|---|
| `cloudflare-project-name` | string | — **(required)** | Cloudflare Pages project name |
| `artifact-name` | string | `'mdbook-site'` | Name of the artifact to download and deploy |
| `directory` | string | `'html'` | Subdirectory within the artifact to deploy |

### 6.4 Secrets

| Secret | Required | Description |
|---|---|---|
| `CLOUDFLARE_ACCOUNT_ID` | Yes | Cloudflare account ID (32 hex chars) |
| `CLOUDFLARE_API_TOKEN` | Yes | Cloudflare API token with Pages edit permission |

Pass both via `secrets: inherit` from the caller (configured as org-level or
repo-level secrets).

### 6.5 Outputs

None.

### 6.6 Job Specification

#### `deploy`

- **Runner**: `ubuntu-latest`
- **Permissions**: none beyond default
- **Steps**:
  1. Download artifact `${{ inputs.artifact-name }}`
  2. `cloudflare/pages-action@v1` with:
     - `apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}`
     - `accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}`
     - `projectName: ${{ inputs.cloudflare-project-name }}`
     - `directory: ${{ inputs.directory }}`
     - `gitHubToken: ${{ github.token }}` (for Cloudflare to post deployment status checks on PRs)

---

## 7. Project-Side PyPI Publish Pattern

PyPI's OIDC trusted publisher mechanism issues tokens per-job. When the publish
step lives inside a reusable workflow in `steamwiz/actions`, the OIDC `sub` claim
encodes `steamwiz/actions` as the repository — not the project being published.
This means a single PyPI trusted publisher entry would grant publish rights for
**all** projects using the shared workflow, which is an unacceptable trust scope.

**The `pypa/gh-action-pypi-publish` step must live in each project's own
`.github/workflows/` directory.** The `steamwiz/actions` library does not include
a publish workflow and never will.

### 7.1 Canonical Project-Side Publish Workflow

Each project creates `.github/workflows/publish.yml` (or inlines the steps into
its main workflow):

```yaml
# .github/workflows/publish.yml
name: Publish to PyPI

on:
  workflow_call:

jobs:
  pypi-publish:
    runs-on: ubuntu-latest
    environment: pypi
    permissions:
      id-token: write   # mandatory for OIDC
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: dist
          path: dist/
      - uses: pypa/gh-action-pypi-publish@release/v1
```

### 7.2 PyPI Trusted Publisher Registration

For each project, add a GitHub Actions trusted publisher on PyPI:

| Field | Value |
|---|---|
| Owner | `steamwiz` |
| Repository | `busy` (the project repo, not `actions`) |
| Workflow filename | `publish.yml` |
| Environment | `pypi` |

### 7.3 GitHub Environment Setup

In the project repo settings, create a GitHub Environment named `pypi`. Optionally
configure required reviewers for additional safety before production PyPI publish.

---

## 8. Release System Design

### 8.1 Tool: python-semantic-release

The `release` job in `python-pipeline.yml` uses
[python-semantic-release](https://python-semantic-release.readthedocs.io/) (PSR).
PSR reads the git commit history since the last tag, determines the next semantic
version from conventional commit messages, updates `pyproject.toml`, creates a git
tag, pushes, and creates a GitHub Release with an auto-generated changelog.

### 8.2 Conventional Commit Format

All Steamwiz projects adopt the
[Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/) specification:

| Commit prefix | Version bump |
|---|---|
| `fix: ...` | patch |
| `feat: ...` | minor |
| `feat!: ...` or `BREAKING CHANGE:` in footer | major |
| `chore:`, `docs:`, `style:`, `refactor:`, `test:`, `build:`, `ci:` | no release |

A release is triggered automatically on every push to `main` **if and only if** the
commit history since the last tag contains at least one `fix:` or `feat:` commit.
Pushes containing only `chore:`/`docs:`/etc. commits produce no release.

### 8.3 Manual Release Override

PSR supports a `--force-level` flag. To trigger a release manually (e.g. to
release accumulated `chore:` commits):

1. Use the GitHub Actions **workflow_dispatch** trigger on the caller's pipeline
   with a `release-level` input (`patch` | `minor` | `major` | `none`).
2. The caller passes `do-release: ${{ inputs.release-level != 'none' }}` to the
   reusable workflow and also sets an env var `PSR_FORCE_LEVEL` that the release
   job reads.

The `python-pipeline.yml` workflow will expose a `release-level` input:

| Input | Type | Default | Description |
|---|---|---|---|
| `release-level` | string | `''` | If set to `patch`, `minor`, or `major`, forces a release at that level regardless of commit messages |

### 8.4 Version in `pyproject.toml`

PSR writes the version directly into `pyproject.toml`. Projects must start with
`version = "0.0.0"`. PSR replaces this
with the computed version and commits the change before tagging.

The `build` job checks out the **tagged commit** (not `main` HEAD) so it builds
the correct versioned package.

### 8.5 Branch Versions (non-main)

On branches other than `main`, no release occurs and no version bump happens. The
`build` job is skipped. Tests and style checks still run.

---

## 9. Secrets and Configuration

### 9.1 Secrets Required Per Project

| Secret | Scope | Description |
|---|---|---|
| `GH_TOKEN` | Org or repo | PAT or fine-grained token with `contents: write`, `issues: write`, `pull-requests: write` for python-semantic-release. Can be the auto `GITHUB_TOKEN` if the repo allows workflow-to-workflow pushes (requires "Allow GitHub Actions to create and approve pull requests" in repo settings). |
| `CLOUDFLARE_ACCOUNT_ID` | Org or repo | Cloudflare account ID |
| `CLOUDFLARE_API_TOKEN` | Org or repo | Cloudflare API token |

`PYPI_ID_TOKEN` does not exist in GitHub Actions. PyPI authentication is handled
entirely via OIDC (`id-token: write` permission), requiring no stored secret.

### 9.2 Recommended Secret Strategy

Configure `CLOUDFLARE_ACCOUNT_ID` and `CLOUDFLARE_API_TOKEN` as
**organisation-level secrets** available to all repos in `steamwiz`. This mirrors
the GitLab group-variable approach.

`GH_TOKEN` can use the default `GITHUB_TOKEN` if repo settings permit; otherwise
use an org-level PAT.

---

## 10. Supported Pipeline Configurations

The following combinations are the intended use cases:

| Project type | Workflows called |
|---|---|
| Python PyPI library + docs | `python-pipeline.yml` + `mdbook-build.yml` + `cloudflare-deploy.yml` + project-side `publish.yml` |
| Python PyPI library, no docs | `python-pipeline.yml` + project-side `publish.yml` |
| Docs/static site only | `mdbook-build.yml` + `cloudflare-deploy.yml` |

---

## 11. What `steamwiz/actions` Does NOT Include

- **PyPI publish step** — per §7, this lives in each project's own workflow
- **SAST / security scanning** — deferred; may be added in a future version
- **Container image build/push** — out of scope for this library
- **Notification steps** (Slack, email, etc.) — out of scope
- **Non-Python language pipelines** — out of scope for v1; may be added later
- **Custom Docker images** — all jobs use standard `ubuntu-latest` runners with
  publicly available GitHub Actions; no private registry needed

---

## 12. Action Pinning Policy

All `uses:` references within `steamwiz/actions` workflows must be pinned to a
specific SHA or major version tag. Use major-version pins (e.g. `@v4`) for
first-party GitHub Actions (`actions/*`) and well-maintained community actions.
Use SHAs for any action with a less certain maintenance track. Never use `@main`
or `@master` in production workflows.

Minimum pinned versions:

| Action | Pin |
|---|---|
| `actions/checkout` | `@v4` |
| `actions/setup-python` | `@v5` |
| `actions/cache` | `@v4` |
| `actions/upload-artifact` | `@v4` |
| `actions/download-artifact` | `@v4` |
| `peaceiris/actions-mdbook` | `@v2` |
| `cloudflare/pages-action` | `@v1` |
| `pypa/gh-action-pypi-publish` | `@release/v1` |
| `EnricoMi/publish-unit-test-result-action` | `@v2` |
| `irongut/CodeCoverageSummary` | `@v1.3.0` |

---

## 13. Permissions Model

Each reusable workflow declares only the permissions it requires. Callers should
pass `secrets: inherit` and rely on org-level or repo-level secrets rather than
passing individual secrets by name where possible.

| Workflow | Permissions declared |
|---|---|
| `python-pipeline.yml` (non-release jobs) | `contents: read` |
| `python-pipeline.yml` (release job) | `contents: write`, `issues: write`, `pull-requests: write` |
| `mdbook-build.yml` | `contents: read` |
| `cloudflare-deploy.yml` | `contents: read` |
| Project-side `publish.yml` | `id-token: write`, `contents: read` |

---

## 14. Repository Bootstrapping

When the `steamwiz/actions` repository is created:

1. Create the repo as **public** under the `steamwiz` GitHub org.
2. Enable "Allow GitHub Actions workflows to create and approve pull requests" in
   repo settings → Actions → General.
3. Add org-level secrets: `CLOUDFLARE_ACCOUNT_ID`, `CLOUDFLARE_API_TOKEN`.
4. Create a `CHANGELOG.md` (empty initially).
5. Create the three workflow files (per this spec).
6. Create an initial git tag `v0.1.0` and the floating tag `v0` once the first
   working version is validated.
7. Update the floating tag `v1` after the first stable release.

The `steamwiz/actions` repo itself uses a minimal self-hosted pipeline (or manual
process) for its own releases; it does not call itself recursively.

## 15. Open Questions and Future Considerations

- **`workflow_dispatch` on the shared pipeline**: The `python-pipeline.yml`
  workflow does not expose `workflow_dispatch` directly (it is `workflow_call`
  only). The caller's workflow can add `workflow_dispatch` with a `release-level`
  input that flows through to the reusable workflow's `release-level` input.
  Each project's caller workflow should document this.

- **Dependabot for action pinning**: The `steamwiz/actions` repo should have a
  Dependabot config (`/.github/dependabot.yml`) to keep action versions current.

- **Test pipeline for the library itself**: Consider a self-test pipeline in
  `steamwiz/actions` that validates workflow YAML syntax (`actionlint`) on every
  push.

- **CodeQL / SAST**: May be added as an optional job in `python-pipeline.yml` in
  a future minor version.

- **Multi-package monorepos**: Out of scope for v1 but the `style-dirs` input
  and the source-dir auto-detection may need extension if a single repo contains
  multiple Python packages.
