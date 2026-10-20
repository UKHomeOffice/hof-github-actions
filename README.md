# HOF Github Actions

This repo is for storing reusable Github actions and workflows for the Home Office Forms (HOF) services.

## Workflows

### Dependabot Auto Merge

Approves eligible Dependabot pull requests and enables squash auto-merge. GitHub
merges only when the repository's required checks and branch rules are satisfied.
Major updates and updates with maintainer changes are not automatically approved.
Dependabot commit verification remains enabled.

Add the following caller workflow to a service's `.github/workflows/` directory:

```yaml
name: Dependabot auto-merge

on:
  pull_request_target:
    types: [opened, reopened, synchronize]

permissions: {}

jobs:
  auto-merge:
    uses: UKHomeOffice/hof-github-actions/.github/workflows/dependabot-auto-merge.yml@main
    secrets:
      DEPENDABOT_APPROVER_APP_ID: ${{ secrets.DEPENDABOT_APPROVER_APP_ID }}
      DEPENDABOT_APPROVER_PRIVATE_KEY: ${{ secrets.DEPENDABOT_APPROVER_PRIVATE_KEY }}
```

The [reusable workflow](.github/workflows/dependabot-auto-merge.yml) runs only for
pull requests authored by `dependabot[bot]` with branches starting with
`dependabot/`, `dependabot-`, or `deps-`. Short `deps-` branches are interpreted as
npm updates at the repository root; services using that prefix must follow this
convention. The workflow checks out only the pinned Dependabot metadata action,
not the pull request's code.

#### Service prerequisites

- Enable auto-merge and configure the required checks and review rules.
- Install the approver GitHub App on the service repository with pull request
  write and contents write permissions.
- Configure `DEPENDABOT_APPROVER_APP_ID` and `DEPENDABOT_APPROVER_PRIVATE_KEY` as
  repository or organization Actions secrets accessible to the caller.
- Allow the service repository to use reusable workflows from this repository
  in the repositories' Actions settings, where required.

#### Rollout and rollback

Publish this workflow to `main` before merging callers that reference it. For
version-controlled rollouts, replace `@main` with a published release tag or full
commit SHA in each caller. Existing inline workflows can coexist with migrated
callers in other services.

To roll back a service, restore its previous inline workflow or point its caller
at a previously verified shared-workflow commit. No data migration is required.
