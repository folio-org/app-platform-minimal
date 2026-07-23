# Feature Build

**Workflow**: `feature-build.yml` (per app repo, from `folio-app-template`)
**Purpose**: Build an application descriptor from a feature branch that includes custom modules built outside the standard registries, without publishing to FAR
**Type**: `workflow_dispatch` wrapper around `application-update.yml`

## Overview

A feature build resolves an application descriptor for a feature branch whose template references
**custom modules** — modules built from their own feature branches, whose descriptors are published
to a private S3 bucket rather than to the FOLIO registry, and whose Docker images live outside
Docker Hub. It:

- resolves those custom module descriptors from an **S3 fallback registry**,
- skips artifact validation for **only** the fallback-resolved modules (primary modules are still
  validated against Docker Hub / npm as usual),
- does **not** publish the descriptor to FAR,
- commits `application.lock.json` back to the feature branch,
- versions the app with the feature branch's **commit hash** (`X.Y.Z-SNAPSHOT.<shortSha>`) instead
  of a build number, so a feature app is distinguishable from a snapshot app in FAR.

All build configuration lives in the feature branch's `pom.xml`. The workflow itself takes no
registry parameters — it only names the branch and the validation switches.

## Configure the feature branch `pom.xml`

Two coupled edits in the `folio-application-generator` plugin section of the app repo's feature
branch:

```xml
<properties>
  <!-- Must be a generator version that supports fallbackModuleRegistries / validateFallbackArtifacts -->
  <folio-application-generator.version>1.5.0-SNAPSHOT</folio-application-generator.version>
</properties>
...
<configuration>
  <templatePath>${templatePath}</templatePath>

  <moduleRegistries>
    <registry>
      <type>okapi</type>
      <url>https://folio-registry.dev.folio.org</url>
    </registry>
  </moduleRegistries>

  <!-- Searched only when a module is not found in moduleRegistries above -->
  <fallbackModuleRegistries>
    <registry>
      <type>s3</type>
      <bucket>eureka-custom-registry</bucket>
      <path>descriptors</path>
    </registry>
  </fallbackModuleRegistries>

  <!-- Custom modules' images are not in Docker Hub, so skip artifact validation for them -->
  <validateFallbackArtifacts>false</validateFallbackArtifacts>

  <awsRegion>us-east-1</awsRegion>
</configuration>
```

Notes:

- **The version bump is required.** Older generator versions (the app poms pin `1.3.0` / `1.4.0`)
  have no fallback-registry support at all. Maven does not fail on `<configuration>` elements a
  plugin does not recognize — it warns and ignores them — so on an old version the
  `<fallbackModuleRegistries>` / `<validateFallbackArtifacts>` elements are silently dropped and no
  custom module is ever resolved from S3. Bump `folio-application-generator.version` to a version
  that supports these elements (`1.5.1-SNAPSHOT` or later).
- **Do not set `<validateArtifacts>false</validateArtifacts>`.** Primary-resolved modules must keep
  their Docker Hub / npm validation. `validateFallbackArtifacts=false` narrows validation to exclude
  only the fallback-resolved custom modules; it does not disable validation.
- `fallbackModuleRegistries` accepts the same registry types as `moduleRegistries`: `s3`, `okapi`,
  `simple`. The S3 registry needs `bucket` and `path`; the region comes from `awsRegion`, not from
  the registry entry.
- Keep this configuration on the feature branch only. A `-SNAPSHOT` generator pin and a custom
  fallback registry must never be merged into `master` or a release branch.

## Run the build

Dispatch **Feature Build** on the app repo, from the Actions tab or the CLI:

```shell
gh workflow run feature-build.yml --repo folio-org/app-<name> \
  -f branch=<feature-branch> \
  -f dry_run=true
```

### Inputs

| Input                        | Description                                                    | Required | Type    | Default   |
|------------------------------|----------------------------------------------------------------|----------|---------|-----------|
| `branch`                     | Feature branch to build the descriptor from                    | Yes      | string  | -         |
| `commit_hash`                | Commit hash used as the descriptor build suffix (empty = auto-detect from the branch HEAD) | No | string | `''` |
| `skip_interface_validation`  | Skip module interface integrity validation                     | No       | boolean | `false`   |
| `skip_dependency_validation` | Dependency validation: `false` / `true` / `bypass`             | No       | choice  | `false`   |
| `dry_run`                    | Build without committing the lock file to the branch           | No       | boolean | `false`   |

A `resolve-build-number` job runs first: it uses `commit_hash` when supplied, otherwise reads the
branch HEAD short SHA (`gh api repos/<repo>/commits/<branch> --jq '.sha[0:7]'`). The resolved value
is passed as `build_number` and becomes the version suffix (`X.Y.Z-SNAPSHOT.<shortSha>`).

The wrapper calls `application-update.yml` with fixed values that make a feature build correct:
`need_pr: false` (commit straight to the feature branch), `publish: false` (never reaches FAR), and
`rely_on_FAR: true` (a feature branch has no matching branch in `platform-lsp`, so dependency
validation resolves against FAR instead of a platform descriptor). Routing through the orchestrator
(`application-update.yml`) rather than the flow directly also produces the run summary and the Slack
notification.

## Related documentation

- [All repository workflows](workflows.md) — overview of the four workflows this repo ships
- [Application Update](https://github.com/folio-org/kitfox-github/blob/master/.github/docs/application-update.md) — the orchestrator this wraps (adds the summary + notification)
- [Application Update Flow](https://github.com/folio-org/kitfox-github/blob/master/.github/docs/application-update-flow.md) — the underlying flow the orchestrator runs
