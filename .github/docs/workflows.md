# Application Repository Workflows

Every `app-*` repository ships four workflows, generated from `folio-app-template`. Each is a thin
wrapper that calls a reusable workflow in
[`kitfox-github`](https://github.com/folio-org/kitfox-github) with `secrets: inherit`; the wrapper
holds only the trigger and the app-specific inputs, while the reusable workflow holds the logic.

| Workflow | Trigger | Calls (kitfox-github) | Purpose |
|---|---|---|---|
| [Application Update Scheduler](#application-update-scheduler) | schedule + manual | `application-update.yml` | Keep module versions current on configured branches |
| [Feature Build](#feature-build) | manual | `application-update.yml` | Build a descriptor from a feature branch with custom modules |
| [Release Application Version](#release-application-version) | manual | `release-application-version-flow.yml` | Create a GitHub Release (tag + descriptor asset) |
| [Release Preparation](#release-preparation) | manual | `release-preparation.yml` | Cut a new release branch from a previous one |

All four resolve the reusable workflow at `@master`, so behavior tracks the latest `kitfox-github`
without changing the app repo.

---

## Application Update Scheduler

**File**: `.github/workflows/update-scheduler.yml`
**Trigger**: `schedule` (every 20 minutes) and `workflow_dispatch`

The automation that keeps applications up to date. It reads `.github/update-config.yml` via the
`get-update-config` action, builds a matrix of enabled branches, and runs `application-update.yml`
for each. Which branches are scanned, whether each uses a PR or a direct commit, and all other
per-branch policy come entirely from `update-config.yml` — not from this workflow.

| Dispatch input | Description | Default |
|---|---|---|
| `dry_run` | Run without creating PRs or committing changes | `false` |

This is the only workflow of the four that runs on its own. The rest are manual.

- Configure it: [update-config.yml schema](https://github.com/folio-org/kitfox-github/blob/master/.github/docs/update-config.md)
- What it runs: [Application Update](https://github.com/folio-org/kitfox-github/blob/master/.github/docs/application-update.md)

## Feature Build

**File**: `.github/workflows/feature-build.yml`
**Trigger**: `workflow_dispatch`

Builds an application descriptor from a **feature branch** whose template references custom modules
built outside the standard registries. It resolves those modules from an S3 fallback registry, skips
artifact validation for only those modules, does not publish to FAR, and commits
`application.lock.json` back to the feature branch. Registry configuration lives in the feature
branch's `pom.xml`; this workflow only names the branch and the validation switches.

| Dispatch input | Description | Default |
|---|---|---|
| `branch` | Feature branch to build from (**required**) | - |
| `commit_hash` | Commit hash used as the version suffix (empty = auto-detect from branch HEAD) | `''` |
| `skip_interface_validation` | Skip module interface integrity validation | `false` |
| `skip_dependency_validation` | `false` / `true` / `bypass` | `false` |
| `dry_run` | Build without committing the lock file | `false` |

The wrapper fixes `need_pr: false`, `publish: false`, and `rely_on_FAR: true` — the combination that
makes a feature build correct (see the flow doc for why) — and versions the app with the feature
branch's commit hash (`X.Y.Z-SNAPSHOT.<shortSha>`). It routes through the orchestrator
`application-update.yml`, so the run also gets a summary and a Slack notification.

- Full pom pattern + details: [Feature Build](feature-build.md)
- What it runs: [Application Update](https://github.com/folio-org/kitfox-github/blob/master/.github/docs/application-update.md)

## Release Application Version

**File**: `.github/workflows/release-application-version.yml`
**Trigger**: `workflow_dispatch`

Tags a commit with the application version, creates a GitHub Release, and attaches
`application-descriptor.json` as a release asset. Normally invoked automatically after a successful
FAR publish (`post-merge-flow.yml`); this manual wrapper exists for re-running or ad-hoc releases.

| Dispatch input | Description | Default |
|---|---|---|
| `commit_sha` | Commit to release (empty = branch HEAD) | `''` |
| `base_branch` | Release branch the commit belongs to (empty = dispatch branch) | `''` |
| `release_notes` | Release notes body text | `''` |
| `dry_run` | Simulate without creating a tag/release | `false` |

- What it runs: [Release Application Version Flow](https://github.com/folio-org/kitfox-github/blob/master/.github/docs/release-application-version-flow.md)

## Release Preparation

**File**: `.github/workflows/release-preparation.yml`
**Trigger**: `workflow_dispatch`

Creates a new release branch (e.g. `R2-2024`) seeded from a previous release branch, pinning module
versions for the new release line. Typically run by the Kitfox team during release cutover.

| Dispatch input | Description | Default |
|---|---|---|
| `previous_release_branch` | Previous release branch, e.g. `R1-2024` (**required**) | - |
| `new_release_branch` | New release branch, e.g. `R2-2024` (**required**) | - |
| `use_snapshot_fallback` | Fall back to `snapshot` if the previous branch is missing | `false` |
| `use_snapshot_version` | Use the snapshot version as the base | `false` |
| `need_pr` | Require a PR for version updates on the new branch | `true` |
| `prerelease_mode` | Module constraints: `false` / `true` / `only` | `'false'` |
| `dry_run` | Run without making changes | `false` |
| `dispatch_id` | Optional identifier for run tracking | `''` |

- What it runs: [Release Preparation Flow](https://github.com/folio-org/kitfox-github/blob/master/.github/docs/release-preparation-flow.md)

---

## Conventions shared by all wrappers

- **`app_name` / `repo`** are derived from the repository (`github.event.repository.name`,
  `github.repository`) — never hardcoded.
- **`secrets: inherit`** passes org/repo secrets through to the reusable workflow; the wrappers
  declare no secrets of their own.
- **`@master` pin** on the `kitfox-github` reusable workflow.
- **`dry_run`** is available on every manual workflow and performs no destructive action (no commit,
  push, PR, tag, or FAR publish).

For setup of a new repository generated from this template, see [SETUP.md](../../SETUP.md).
