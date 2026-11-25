# collections-standards

Reusable GitHub Actions workflows that standardize CI across Ansible collections.
Each collection keeps a small caller workflow that owns its triggers, while the jobs,
version matrices and pinned actions live here in one place.

| Workflow | Purpose | Status check for branch protection |
|---|---|---|
| [`changelog.yml`](.github/workflows/changelog.yml) | Require a changelog fragment | `<job> / changelog / Require a changelog` |
| [`units.yml`](.github/workflows/units.yml) | `ansible-test units` across ansible-core/Python versions, coverage artifact | `<job> / unit-check` |
| [`sonarcloud.yml`](.github/workflows/sonarcloud.yml) | Fork-safe SonarCloud analysis using the units coverage | SonarCloud quality gate (posted by SonarCloud) |
| [`certification.yml`](.github/workflows/certification.yml) | Automation Hub certification checks plus sanity for non-AAP ansible-core versions | `<job> / certification-check` |
| [`docs-lint.yml`](.github/workflows/docs-lint.yml) | `antsibull-docs lint-collection-docs` | `<job> / Lint extra docsite docs and links` |

`<job>` is the job ID in the caller workflow, for example `units / unit-check`.

## Usage

Copy the callers you need from [`examples/`](examples/) into the collection's
`.github/workflows/` directory and replace the placeholder SHA with a release commit:

```yaml
jobs:
  units:
    uses: eco-ansible-content/collections-standards/.github/workflows/units.yml@<release commit SHA> # v1.0.0
```

Pin to a full commit SHA with the version as a comment, and enable Dependabot for the
`github-actions` ecosystem in the collection repository so new releases arrive as PRs:

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
```

The caller workflow decides **when** to run (triggers, branches, schedule) and
**concurrency**. The reusable workflow decides **what** runs. None of the reusable
workflows take inputs, so every collection runs the same checks. Workflows with several
jobs (units, certification) end with an [alls-green](https://github.com/re-actors/alls-green)
job that gives a single, stable status check to require in branch protection.

### Repository requirements

- `galaxy.yml` at the repository root. Units installs the collection and its dependencies
  from it, so `build_ignore` must not exclude `tests/unit`.
- `changelogs/config.yaml` for changelog fragment validation, and a `skip-changelog`
  label for pull requests that do not need a fragment.
- `sonar-project.properties` on the **default branch** for SonarCloud. Start from
  [`examples/sonar-project.properties`](examples/sonar-project.properties).
- The units caller workflow must be named `Units`, because SonarCloud is triggered by
  workflow name and requires its `units-coverage` artifact.

## Supported versions

The tested ansible-core versions are defined here and are the same for every
collection; they cannot be overridden by callers.

`devel` is allowed to fail.

When new ansible-core versions are announced in
[news-for-maintainers](https://github.com/ansible-collections/news-for-maintainers),
update the matrices in `units.yml` and `certification.yml`, then cut a release.

## Releasing

1. Merge changes to `main` with CI passing.
2. Tag a semantic version (`vX.Y.Z`) and create a GitHub release. Bump the major version
   for breaking changes, such as renamed workflows or status checks, new required secrets,
   or new repository requirements.
3. Dependabot in collection repositories proposes the new SHA.

This repository must stay **public** so repositories in other organizations can call
its workflows.

## Development

[`gh-workflow-lint.yml`](.github/workflows/gh-workflow-lint.yml) is the only workflow
that runs on this repository itself; every other workflow is reusable. It runs [actionlint](https://github.com/rhysd/actionlint)
and [zizmor](https://docs.zizmor.sh) on the reusable workflows and the examples. To run
them locally:

```shell
actionlint .github/workflows/*.yml examples/*.yml
zizmor .github
zizmor --offline examples/*.yml
```

All third-party actions and workflows are pinned to full commit SHAs and updated by
Dependabot. The exception is `ansible-network/github_actions` in `changelog.yml`, which has
no release tags, so its pin must be updated by hand. Keep
logic inline in the workflow files (no separate scripts or local composite actions) so
that pinning a workflow SHA pins everything it runs.
