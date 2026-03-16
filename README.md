# platform-ci

Reusable GitHub workflows shared across AnthusAI repositories.

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

Bootstrap mode is intentionally conservative:

- PRs to `main` run `check`.
- `apply` runs only via manual `workflow_dispatch` (`mode=apply`).

After initial validation, you can enable automatic `apply` on push to `main`.
