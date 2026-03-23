# platform-ci

Reusable GitHub workflows shared across AnthusAI repositories.

## Reusable semantic-release workflow

Workflow path:

- `.github/workflows/semantic-release-reusable.yml`

It standardizes protected-branch release automation using anthusbot credentials
and emits release outputs for downstream publish jobs.

### Required caller secret

- `ANTHUSBOT_GH_TOKEN`

Optional caller secret:

- `PYPI_TOKEN` (only needed when semantic-release config/publish path uses it)

### Caller workflow example

```yaml
jobs:
  release:
    uses: AnthusAI/platform-ci/.github/workflows/semantic-release-reusable.yml@main
    with:
      python_version: "3.12"
      checkout_ref: ${{ github.sha }}
      release_branch: main
      install_command: 'pip install "python-semantic-release~=10.0"'
      run_publish: true
    secrets:
      anthusbot_gh_token: ${{ secrets.ANTHUSBOT_GH_TOKEN }}
      semantic_release_pypi_token: ${{ secrets.PYPI_TOKEN }}
```

Provided outputs:

- `released` (`true`/`false`)
- `tag` (e.g. `v1.2.3`)
- `version` (e.g. `1.2.3`)

## Reusable coverage badge workflow

Workflow path:

- `.github/workflows/coverage-badge-reusable.yml`

It standardizes coverage badge updates by consuming a coverage JSON artifact from a
completed CI run and committing badge changes with anthusbot credentials.

### Required caller secret

- `ANTHUSBOT_GH_TOKEN`

### Caller workflow example

```yaml
name: Coverage Badge

on:
  workflow_run:
    workflows: ["Continuous integration"]
    types: [completed]

permissions:
  actions: read
  contents: write

jobs:
  update-coverage-badge:
    if: ${{ github.event.workflow_run.conclusion == 'success' && github.event.workflow_run.event == 'push' && github.event.workflow_run.head_repository.full_name == github.repository && github.event.workflow_run.head_branch == 'main' && github.event.workflow_run.actor.login != 'anthusbot' }}
    uses: AnthusAI/platform-ci/.github/workflows/coverage-badge-reusable.yml@main
    with:
      checkout_ref: ${{ github.event.workflow_run.head_sha }}
      release_branch: "main"
      source_workflow_run_id: ${{ github.event.workflow_run.id }}
      source_artifact_name: coverage-json-py312
      coverage_json_path: coverage.json
      badge_output_path: coverage_badge.json
      badge_label: coverage
      commit_message: "chore: update coverage badge"
    secrets:
      anthusbot_gh_token: ${{ secrets.ANTHUSBOT_GH_TOKEN }}
```

### Expected CI artifact in caller repo

Caller CI should upload a coverage JSON artifact on the run this workflow listens to.
Example producer step:

```yaml
- name: Export coverage JSON
  if: matrix.python-version == '3.12'
  run: python -m coverage json -o coverage.json

- name: Upload coverage artifact
  if: matrix.python-version == '3.12'
  uses: actions/upload-artifact@v4
  with:
    name: coverage-json-py312
    path: coverage.json
```

Provided outputs:

- `updated` (`true`/`false`)
- `coverage_percent` (rounded integer percent as string)

## Reusable ruleset sync workflow

Workflow path:

- `.github/workflows/ruleset-sync-reusable.yml`

It manages GitHub repository rulesets from JSON desired-state files stored in the caller repo.

### Required caller secret

Create an Actions repository secret in each caller repo:

- `RULESET_ADMIN_TOKEN`

Recommended token type and permissions:

- Fine-grained PAT
- Repository access: only selected repositories
- Repository permissions:
  - Administration: Read and write
  - Metadata: Read-only

### Caller workflow example

```yaml
name: Ruleset Sync

on:
  pull_request:
    branches: [main]
    paths:
      - 'infra/github-rulesets/**'
      - '.github/workflows/ruleset-sync.yml'
  push:
    branches:
      - main
      - 'feature/**'
    paths:
      - 'infra/github-rulesets/**'
      - '.github/workflows/ruleset-sync.yml'
  workflow_dispatch:
    inputs:
      mode:
        description: 'Run mode'
        required: true
        default: check
        type: choice
        options: [check, apply]

jobs:
  check:
    if: ${{ github.event_name != 'workflow_dispatch' || inputs.mode == 'check' }}
    uses: AnthusAI/platform-ci/.github/workflows/ruleset-sync-reusable.yml@main
    with:
      mode: check
      ruleset_glob: infra/github-rulesets/*.json
    secrets:
      ruleset_admin_token: ${{ secrets.RULESET_ADMIN_TOKEN }}

  apply:
    if: ${{ (github.event_name == 'push' && github.ref == 'refs/heads/main') || (github.event_name == 'workflow_dispatch' && inputs.mode == 'apply') }}
    uses: AnthusAI/platform-ci/.github/workflows/ruleset-sync-reusable.yml@main
    with:
      mode: apply
      ruleset_glob: infra/github-rulesets/*.json
    secrets:
      ruleset_admin_token: ${{ secrets.RULESET_ADMIN_TOKEN }}
```

### Desired-state file format

Caller repos store JSON files under `infra/github-rulesets/` with at least:

- `name`
- `target`
- `enforcement`
- `conditions`
- `rules`

The workflow resolves rulesets by `name` + `target`, updates if found, and creates if missing in `apply` mode.

## This repo's own ruleset management

`platform-ci` also manages its own rulesets using the same reusable workflow:

- Desired state: `infra/github-rulesets/main-branch-protection.json`
- Wrapper workflow: `.github/workflows/ruleset-sync.yml`

Steady-state mode:

- PRs to `main` run `check`.
- Pushes to `main` run `apply`.
- Manual `workflow_dispatch` supports both `check` and `apply`.
