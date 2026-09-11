# Changelog

Notable changes to the reusable workflows in this repository.

Callers pin a tag of this repository as a whole, so a release describes every
workflow at once. Read the entry for the version you are moving to, and every
entry between it and the version you are on. Anything marked **Breaking**
requires a change in the calling repository; everything else is safe to adopt by
bumping the tag.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this repository uses [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
loosely: while the major version is 0, a breaking change bumps the minor.

## [0.4.0] — 2026-09-11

**Breaking — the workflow files are now `.yaml`, not `.yml`.** Update the path
together with the tag:

```diff
-    uses: fmidev/.github/.github/workflows/openshift-deploy.yml@0.3.3
+    uses: fmidev/.github/.github/workflows/openshift-deploy.yaml@0.4.0
```

A caller that bumps the tag without renaming gets "workflow was not found" and
nothing runs. Callers pinned to `@0.3.3` are unaffected.

**Breaking — `openshift-deploy` no longer declares
`HELM_REPOSITORY_PRIVATE_DEPLOY_KEY`.** It was required but never used. Callers
using `secrets: inherit` need no change; a caller passing secrets explicitly must
drop that line, or the call fails with "invalid input".

**Breaking — `docker-build-and-push` no longer declares `CREATE_LATEST_TAG`.** It
was declared but never implemented. A caller passing it fails with "invalid
input"; drop the line.

**Breaking — `DEPLOY_CONFIG_REPOSITORY_BRANCH` is now honored.** It was declared
but silently ignored, so every deployment went to the GitOps repository's
default branch regardless. A caller that sets it should check the value names a
branch Argo CD actually reconciles before upgrading — a wrong branch now
produces a green run that deploys nothing.

**Breaking — a version carrying SemVer build metadata is rejected up front.**
`1.2.3+hotfix` previously passed validation, published the Helm chart, and then
failed the image push, since `+` cannot appear in an image tag. It now fails
before anything is published.

**Breaking — `deploy-via-rsync` and `deploy-to-fmi-openshift` are removed.** Both
were obsolete and uncalled, and there is no replacement for either at this
version. `deploy-to-fmi-openshift` was the original single-job pipeline,
superseded by `openshift-deploy` together with `openshift-promote`.

### Added

- `openshift-promote` deploys an already-published version to further
  environments without rebuilding it, after verifying that both the container
  image and the Helm chart for that version exist in the registry. Intended for
  environments that must not deploy automatically on a tag push, such as
  production.
- `post-to-slack-deploy-automation-notifier` posts a plain text message to the
  deploy automation Slack channel. A promotion announces itself through it when
  the optional `SLACK_WEBHOOK_URL` secret is available.
- `lint` is this repository's own CI, and the first it has had: actionlint,
  yamllint `--strict` and zizmor, each in a container pinned by digest. Not
  callable.
- New optional inputs, all defaulting to the previous behaviour:
  `IMAGE_TAG_PREFIX`, `IMAGE_TAG_SUFFIX`, `IMAGE_PLATFORMS`, `IMAGE_FILE`,
  `HELM_CHART_PATH`, `HELM_VERSION`, `DEPLOY_CONFIG_REPOSITORY_SERVER_URL`, and
  the `IMAGE_BUILD_ARGS` secret. `HELM_CHART_KEY` names the top-level key in the
  GitOps values.yaml, and on `openshift-promote` is separate from
  `HELM_CHART_NAME`, the published chart's registry name — set both when an
  umbrella chart references the application chart under a dependency alias.
- A build summary naming the image that was pushed, and a promotion summary
  naming the version, environments, image and chart.

### Fixed

- The SemVer validation in `openshift-deploy` checked `$?` after a pipeline under
  `bash -e`, so it could never print its own error message.
- An empty or separator-only `DEPLOY_CONFIG_REPOSITORY_ENVIRONMENTS` updated
  nothing and reported success. It is now an error, as is an environment
  directory that does not exist in the GitOps repository.
- `yq` creates whatever path it is given, so a values key that did not match what
  the GitOps repository actually holds appended a block nothing reads while
  leaving the real image tag untouched — a green deployment that deployed
  nothing. The full `<key>.application.deployment.image.tag` path and the chart
  dependency must now already exist.
- An image name with an uppercase letter — the default is the repository's name,
  which GitHub does not lowercase — reached the registry before failing. It is
  now rejected before the registry login.
- A rejected push to the GitOps repository is rebased and retried, rather than
  failing the deployment. Several applications share one GitOps repository, so
  simultaneous deployments race on the push.
- Re-deploying a version already referenced in GitOps is a successful no-op
  rather than a failure.
- `update-openshift-application-version` and
  `helm-build-and-publish-to-github` could not commit from a manually dispatched
  run, because `github.event.head_commit` does not exist outside a push event.
- A build context in a subdirectory looked for its Containerfile in the
  repository root.

### Changed

- Every external action is pinned by commit SHA, with the version in a trailing
  comment, and Dependabot is configured to bump both together on a weekly
  schedule with a seven-day cooldown.
- Runners are pinned to `ubuntu-24.04` instead of `ubuntu-latest`, and the two
  jobs still on `ubuntu-22.04` are moved off it ahead of its deprecation.
  `ubuntu-26.04` is deliberately not used while it remains a public preview
  outside the Actions SLA.
- Helm is pinned to 3.22.0, and is now an input.
- Values that reach a shell are passed through the environment rather than
  interpolated into the script.
- Registry credentials are given to `helm registry login` on stdin rather than on
  the command line.
- `README.md` documents the calling conventions, the layout the GitOps repository
  must have, and the release process.

### Removed

- `update-openshift-application-version` installed Helm and never used it.
- `docker-build-and-push` pulled in a third-party action only to compute a short
  commit SHA.

## Earlier releases

Reconstructed from git history; these releases predate this changelog and the
entries may be incomplete.

### [0.3.3] — 2024-03-01

- Fixed a variable reference in `openshift-deploy`.

### [0.3.2] — 2024-02-29

- Added `APPLICATION_BUILD_CONTEXT` to `openshift-deploy`, making the container
  build context configurable.

### [0.3.1] — 2024-02-28

- Pinned `azure/setup-helm` to an explicit version.

### [0.3.0] — 2024-02-28

- Updated the sub-workflow references and dropped obsolete workflows.

### [0.2.3] — 2024-01-09

- Fixed variable expansion in the GitOps update step.

### [0.2.2] — 2024-01-09

- Fixed an input name and its default value.

### [0.2.1] — 2023-09-15

- Added `deploy-via-rsync`.

### [0.2.0] — 2023-05-09

- **Breaking:** `DEPLOY_CONFIG_REPOSITORY_ENVIRONMENT` became
  `DEPLOY_CONFIG_REPOSITORY_ENVIRONMENTS` and takes a comma-separated list, so
  one deployment can update several environments.

### [0.1.6] — 2022-09-13

- Sub-workflow references pinned to a tag instead of a branch.

### [0.1.5] — 2022-09-05

- Added `helm-build-and-publish-to-github`, publishing charts to the git-based
  chart repository.

### [0.1.4] — 2022-05-20

- The default image tag became `latest`, and the registry is included in the
  pushed reference.

### [0.1.3] — 2022-04-26

- **Breaking:** the Helm chart is read from `conf/helm/` instead of `deploy/`.

### [0.1.2] — 2022-04-08

- First `README.md`.

### [0.1.1] — 2022-03-28

- Only secrets are required; the remaining values became optional inputs.

### [0.1.0] — 2022-03-28

- First release: build, publish and deploy to OpenShift through the GitOps
  repository.
