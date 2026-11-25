# CI Compliance Standards — Collection Developer Guide

Audience: collection maintainers. This tells you what your repo must contain to be reported as compliant by the site that scans GitHub repos, and shows the minimal setup that satisfies each standard.

## Standards

| ID | Applies to | Level | Requirement |
| :---- | :---- | :---- | :---- |
| S1 | all repos | required | CI calls the shared workflows defined in this repo — certification, changelog and docs-lint for every repo; units and sonarcloud additionally for python repos |
| S2 | all repos | required | Run integration tests in CI |
| S3 | all repos | required | Use Dependabot for GitHub Actions updates |
| S4 | all repos | required | Declare a minimum supported ansible-core version at or above the current floor — `requires_ansible` in `meta/runtime.yml` must be **>= 2.16** at time of writing (the floor changes over time; check the dashboard site) |

Windows repos (PowerShell module collections) are exempt from the unit-test-related parts of S1 — the units and sonarcloud workflows. Everything else applies to both groups.

---

## S1 — Shared workflows

Compliance for S1 is based on one thing: your repository's CI calls the shared reusable workflows defined in this repo. The scanner looks for jobs whose `uses:` points at them:

```yaml
jobs:
  units:
    uses: eco-ansible-content/collections-standards/.github/workflows/units.yml@<release commit SHA> # vX.Y.Z
```

| Workflow | Purpose | Required for | Status check for branch protection |
|---|---|---|---|
| [`certification.yml`](.github/workflows/certification.yml) | Automation Hub certification checks plus sanity for non-AAP ansible-core versions | all repos | `<job> / certification-check` |
| [`units.yml`](.github/workflows/units.yml) | `ansible-test units` across ansible-core/Python versions, coverage artifact | python repos | `<job> / unit-check` |
| [`changelog.yml`](.github/workflows/changelog.yml) | Require a changelog fragment | all repos | `<job> / changelog / Require a changelog` |
| [`sonarcloud.yml`](.github/workflows/sonarcloud.yml) | Fork-safe SonarCloud analysis using the units coverage | python repos | SonarCloud quality gate (posted by SonarCloud) |
| [`docs-lint.yml`](.github/workflows/docs-lint.yml) | `antsibull-docs lint-collection-docs` | all repos | `<job> / Lint extra docsite docs and links` |

Setting a workflow up is always the same three steps:

1. Copy the caller from [`examples/`](examples/) into the collection's `.github/workflows/` directory.
2. Replace the placeholder SHA with a release commit SHA, keeping the version comment.
3. Enable Dependabot (S3) so new releases of the shared workflows arrive as pull requests.

The caller decides **when** the workflow runs (triggers, branches, schedule, concurrency); the shared workflow decides **what** runs. None of them take inputs, so every collection runs the same checks. Homegrown jobs that merely do something similar no longer pass — the shared workflows are the standard, and the jobs, version matrices and pinned actions are maintained here in one place.

### Certification

```yaml
# .github/workflows/certification.yml — trimmed from examples/certification.yml
name: Certification
on:
  pull_request:
    branches: [main]
  schedule:
    - cron: "0 6 * * *" # nightly run catches upstream lint and sanity changes
  workflow_dispatch:

jobs:
  certification:
    uses: eco-ansible-content/collections-standards/.github/workflows/certification.yml@<release commit SHA> # vX.Y.Z
```

The shared workflow runs the Automation Hub certification checks — build/import, ansible-lint and sanity for the ansible-core versions shipped with AAP (currently stable-2.16, stable-2.18, stable-2.20), skipping versions older than `requires_ansible` in `meta/runtime.yml` — plus sanity tests for the remaining non-AAP ansible-core versions (currently 2.19, 2.21, 2.22 and devel).

Notes:

- Extra jobs in the same file are fine — they don't affect this check.
- Files that are not related to the core functionality of the collection can cause ansible-lint to fail. If this happens, add an `.ansible-lint` file that excludes them.
- Sanity failures that cannot be fixed and are allowed to be ignored go into the `tests/sanity/ignore-<version>.txt` file of each affected ansible-core version.
- Status check for branch protection: `<job> / certification-check`, for example `certification / certification-check` — a single alls-green job covers both the certification and sanity jobs.

### Units

```yaml
# .github/workflows/units.yml — trimmed from examples/units.yml
name: Units # must stay "Units": the SonarCloud workflow is triggered by name
on:
  push:
    branches: [main, stable-*]
  pull_request:

jobs:
  units:
    uses: eco-ansible-content/collections-standards/.github/workflows/units.yml@<release commit SHA> # vX.Y.Z
```

The shared workflow installs the collection with `ansible-galaxy collection install .` (including its dependencies) and runs `ansible-test units --docker` across the ansible-core/Python matrix defined here, publishing a coverage artifact that the SonarCloud workflow consumes.

Notes:

- Requires `galaxy.yml` at the repository root. The tests run from the installed copy, so `build_ignore` must not exclude `tests/unit`.
- The ansible-core/Python matrix is defined here and cannot be overridden by callers — currently **2.16, 2.18, 2.19, 2.20, 2.21, 2.22**, each paired with the oldest and newest controller Python it supports, plus `devel` which is allowed to fail. When new ansible-core versions are announced in [news-for-maintainers](https://github.com/ansible-collections/news-for-maintainers), the matrix is updated here and a release is cut; Dependabot (S3) then proposes the new SHA in your repository. Merging those PRs promptly is what keeps your version coverage current.
- Status check for branch protection: `<job> / unit-check`, for example `units / unit-check`.
- Your own `make units` or `ansible-test units` jobs no longer pass this standard — replace them with the caller above.

### Changelog

```yaml
# .github/workflows/changelog.yml — trimmed from examples/changelog.yml
name: Changelog
on:
  pull_request:
    # labeled/unlabeled re-run the check when the skip-changelog label changes.
    types: [opened, reopened, labeled, unlabeled, synchronize]
    branches: [main, stable-*]

jobs:
  changelog:
    uses: eco-ansible-content/collections-standards/.github/workflows/changelog.yml@<release commit SHA> # vX.Y.Z
```

Notes:

- Requires `changelogs/config.yaml` in the repository, and a `skip-changelog` label for pull requests that do not need a fragment.
- The trigger types above are required, so adding or removing the label re-evaluates the check.
- The shared workflow wraps the ansible-network changelog workflow; that inner pin is maintained here.
- Status check for branch protection: `<job> / changelog / Require a changelog`, for example `changelog / changelog / Require a changelog`.

### SonarCloud

```yaml
# .github/workflows/sonarcloud.yml — trimmed from examples/sonarcloud.yml
name: SonarCloud
on:
  # workflow_run is required to access SONAR_TOKEN for fork pull requests.
  workflow_run:
    workflows: [Units] # must match the units caller's name
    types: [completed]

jobs:
  sonarcloud:
    uses: eco-ansible-content/collections-standards/.github/workflows/sonarcloud.yml@<release commit SHA> # vX.Y.Z
    secrets:
      SONAR_TOKEN: ${{ secrets.ANSIBLE_COLLECTIONS_ORG_SONAR_TOKEN_CICD_BOT }}
```

Notes:

- Depends on the units caller: it analyses the coverage produced by the Units workflow, so that caller must be named `Units`.
- Requires `sonar-project.properties` on the **default branch**, and the `SONAR_TOKEN` secret.
- Fork-safe: it runs in the context of the default branch, takes `sonar-project.properties` from there, and never executes code from the analysed commit.
- `workflow_run` always uses this file from the default branch, so changes to it only take effect after they are merged.
- Status check for branch protection: the SonarCloud quality gate, posted by SonarCloud.

### Docs lint

```yaml
# .github/workflows/docs-lint.yml — trimmed from examples/docs-lint.yml
name: Docs lint
on:
  push:
    branches: [main, stable-*]
  pull_request:
  schedule:
    - cron: "0 6 * * *"

jobs:
  docs-lint:
    uses: eco-ansible-content/collections-standards/.github/workflows/docs-lint.yml@<release commit SHA> # vX.Y.Z
```

The shared workflow runs `antsibull-docs lint-collection-docs` with plugin docs and link checks. Status check for branch protection: `<job> / Lint extra docsite docs and links`, for example `docs-lint / Lint extra docsite docs and links`.

## S2 — Integration tests (all repos)

There is no shared integration workflow, so this standard keeps its previous recognition rules. The scanner recognizes any of: `ansible-test-gh-action` with `testing-type: integration`, the `ansible/ansible-content-actions` integration reusable workflow, a `make integration` step, or a direct `ansible-test integration` command. Minimal example:

```yaml
# .github/workflows/integration.yml
name: Integration
on:
  pull_request:
  push:
    branches: [main, stable-*]

jobs:
  integration:
    name: I (Ⓐ${{ matrix.ansible }})
    strategy:
      matrix:
        ansible: [stable-2.20, stable-2.21, devel]
    runs-on: ubuntu-latest
    steps:
      - name: Perform integration testing
        uses: ansible-community/ansible-test-gh-action@release/v1
        with:
          ansible-core-version: ${{ matrix.ansible }}
          testing-type: integration
```

Any shape that ends up running `ansible-test integration` (directly or via `make integration`) is recognized — microsoft.wsl's Azure Windows-targeted job and microsoft.mecm's self-hosted job both pass because they invoke `make integration`. There is **no required ansible-core version set for integration**.

## S3 — Dependabot

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

The scanner requires a `github-actions` ecosystem entry; any schedule interval passes. Dependabot is also what keeps your shared-workflow pins (S1) current — without it your repository drifts out of date as new releases are cut here.

## S4 — Minimum supported ansible-core version

The scanner reads `requires_ansible` from `meta/runtime.yml` and checks it against the current floor, **2.16** at time of writing. The floor changes over time; check the dashboard site.

```yaml
# meta/runtime.yml
---
requires_ansible: ">=2.16.0"
```

Notes:

- Any value at or above the floor passes; a value below it fails.
- The floor matches the oldest ansible-core version tested by the shared units workflow, and the certification checks skip ansible-core versions older than `requires_ansible` — declaring the floor keeps everything aligned.

---

## Migrating from the old checks

The standards were renumbered. Certification, units (including its ansible-core version coverage), changelog, SonarCloud and docs lint are now all part of S1. The old integration (S4) and Dependabot (S7) standards are now S2 and S3, and the minimum ansible-core version (S4) is new.

If your repository currently passes the old rules by calling the underlying workflows directly (the partner certification checker, the ansible-network changelog workflow) or by running its own units jobs, replace those jobs with the callers from [`examples/`](examples/). The shared workflows run the same checks — only where the jobs, matrices and pins are defined changes. Keep your integration workflow (S2) as is.

## What a fully compliant repo looks like

Python collection:

```
.github/
  dependabot.yml            # S3
  workflows/
    certification.yml       # S1
    units.yml               # S1
    integration.yml         # S2 — your own integration workflow
    changelog.yml           # S1
    sonarcloud.yml          # S1
    docs-lint.yml           # S1
meta/runtime.yml            # S4 — requires_ansible >= 2.16
```

Windows collection:

```
.github/
  dependabot.yml            # S3
  workflows/
    certification.yml       # S1
    integration.yml         # S2
    changelog.yml           # S1
    docs-lint.yml           # S1
meta/runtime.yml            # S4
```

Quick self-check against each standard:

| Standard | The scanner passes you if… |
| :---- | :---- |
| S1 | your CI has jobs `uses:` each shared workflow — certification, changelog and docs-lint (all repos), plus units and sonarcloud (python repos) |
| S2 | a job runs integration via ansible-test-gh-action (`testing-type: integration`), the content-actions integration workflow, `make integration`, or `ansible-test integration` |
| S3 | `.github/dependabot.yml` exists with a `github-actions` ecosystem entry |
| S4 | `requires_ansible` in `meta/runtime.yml` is at or above the current floor (2.16 at time of writing) |
