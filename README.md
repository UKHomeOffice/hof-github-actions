# HOF Github Actions

This repo is for storing reusable Github actions and workflows for the Home Office Forms (HOF) services.

## Contents

- [Workflows](#workflows)
  - [Dependabot Auto Merge](#dependabot-auto-merge)
    - [Service prerequisites](#service-prerequisites)
    - [Rollout and rollback](#rollout-and-rollback)
  - [Sync hof-nodejs image in .drone.yml](#sync-hof-nodejs-image-in-droneyml)
    - [Service prerequisites](#service-prerequisites-1)

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

### Sync hof-nodejs image in .drone.yml

Copies the pinned `hof-nodejs` image from a Dependabot pull request's `Dockerfile`
to the `node_image_name` anchor in `.drone.yml`, and pushes the change to the pull
request branch as a signed commit.

Add the following caller workflow to a service's `.github/workflows/` directory:

```yaml
name: Sync hof-nodejs image in .drone.yml

# pull_request_target is needed for access to Actions secrets on Dependabot PRs; PR code is only read, never executed.
on:
  pull_request_target:
    types: [opened, reopened, synchronize]
    paths:
      - Dockerfile

permissions: {}

concurrency:
  group: sync-drone-node-image-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  sync:
    permissions:
      contents: read
    uses: UKHomeOffice/hof-github-actions/.github/workflows/sync-drone-node-image.yml@main
    secrets:
      app-id: ${{ secrets.DEPENDABOT_APPROVER_APP_ID }}
      private-key: ${{ secrets.DEPENDABOT_APPROVER_PRIVATE_KEY }}
```

The [reusable workflow](.github/workflows/sync-drone-node-image.yml) runs only for
`pull_request_target` events on pull requests opened and updated by
`dependabot[bot]` from a branch in the same repository. It does not execute the
pull request's code; it only reads `Dockerfile` and edits `.drone.yml`.

#### Service prerequisites

- `Dockerfile` must reference the image as
  `quay.io/ukhomeofficedigital/hof-nodejs:<tag>@sha256:<digest>`.
- `.drone.yml` must define the image on a line starting with
  `node_image_name: &node_image_name `.
- The calling job must grant `contents: read`, as reusable workflows cannot
  exceed the caller's permissions.
- Install the GitHub App on the service repository with contents write
  permission, and make its ID and private key available to the caller as Actions
  secrets.

Once the App commits to a Dependabot branch, Dependabot no longer rebases that
pull request automatically. Comment `@dependabot recreate` to regenerate it; the
sync then runs again.
