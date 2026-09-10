# FMI GitHub Actions workflow templates and starter workflows

Shared reusable workflows for the `fmidev` organisation on github.com.

## Available workflows

| Workflow | Purpose |
|---|---|
| `openshift-deploy.yaml` | Build, publish and deploy the triggering ref. Normally called on a tag push. |
| `openshift-promote.yaml` | Deploy an **already published** version to further environments. Nothing is rebuilt. |
| `docker-build-and-push.yaml` | Build a container image and push it to the registry. |
| `helm-build-and-publish-to-oci-repo.yaml` | Package a Helm chart and push it to the OCI registry. |
| `helm-build-and-publish-to-github.yaml` | Package a Helm chart and commit it to the git-based chart repository. |
| `update-openshift-application-version.yaml` | Rewrite the version references in the OpenShift GitOps repository. |
| `post-to-slack-deploy-automation-notifier.yaml` | Post a plain text message to the deploy automation Slack channel. |

`lint.yaml` is the exception: it is this repository's own CI and is not meant to
be called.

Callers pin a tag of this repository, for example
`uses: fmidev/.github/.github/workflows/openshift-deploy.yaml@0.4.0`.

## Relationship to `pak/.github`

The FMI GitHub Enterprise instance carries the same set of workflows in
`pak/.github`, and the two are maintained together: same GitOps repository, same
registry, same lint configuration, and deliberately the same file and input
names wherever both instances can support them.

They are not interchangeable, because the instances differ:

| | this repository | `pak/.github` (GHES) |
|---|---|---|
| Runners | GitHub-hosted `ubuntu-24.04` | self-hosted `docker` / `podman` |
| `actions/*` | upstream, any released version | resolved against the instance's own mirror, currently capped at `actions/checkout` v6.0.2 |
| Slack | `slackapi/slack-github-action` v4, `webhook` input | held at v1.24.0, `SLACK_WEBHOOK_URL` environment variable |
| Chart publishing | OCI registry, or the git-based chart repository | OCI registry only |

A change made here usually belongs there too, but the pins and the runner labels
never copy across unchanged.

## Checks on a change here

Nothing in this repository is built or tested by the repository itself. Every
workflow runs on behalf of some other repository, so a mistake lands on whoever
called it, part-way through a deployment. `lint.yaml` is the only gate before
that, and all three of its jobs must stay green:

- **actionlint** for expressions that do not parse, unknown contexts, actions on
  retired runners, and shellcheck findings in `run:` scripts.
- **yamllint** `--strict`, configured in `.yamllint.yaml`, for the defects that
  change meaning: a duplicate key silently discarding a value, a tab, broken
  indentation.
- **zizmor** for the failure modes specific to shared CI: an untrusted value
  reaching a shell, a mutable action reference, a credential left behind in the
  workspace. Accepted findings are listed with their reasons in
  `.github/zizmor.yml`; the audit is expected to report nothing at all.

All three run in containers pinned by digest, so a lint result is reproducible
and identical to the one `pak/.github` produces on its self-hosted runners.
Dependabot keeps the actions current but cannot see a container named inside a
`run:` step, so bump those digests along with the tag.

## Pinning

Every external action is pinned by SHA, with the version in a trailing comment
so it stays readable and Dependabot can bump both together:

```yaml
uses: docker/login-action@dbcb813823bdd20940b903addbd779551569679f # v4.6.0
```

Reusable workflows from this repository stay on their release tag, so that one
tag describes the whole set. That exception is encoded as a policy in
`.github/zizmor.yml` rather than left to reviewer memory.

Runners are pinned to `ubuntu-24.04` rather than left on `ubuntu-latest`, so that
a GitHub-side image migration cannot change what thirty-odd repositories deploy
with, without a commit here. `ubuntu-24.04` is the newest generally available
image; `ubuntu-26.04` exists but is a public preview and is explicitly outside
the Actions SLA, which is not a suitable base for a deployment pipeline. Revisit
when it goes GA.

Helm is pinned to a 3.x release. Helm 4 is out, and the GitHub-hosted images
already ship it, but the charts in the GitOps repository and the Argo CD instance
that reconciles them are Helm 3; moving is a separate, deliberate change.

## Deploying on a tag push

`openshift-deploy.yaml` builds the image and the Helm chart, then updates the
version references in the GitOps repository for every environment listed in
`DEPLOY_CONFIG_REPOSITORY_ENVIRONMENTS`:

```yaml
on:
  push:
    tags:
      - '[0-9]+.[0-9]+.[0-9]+'

permissions:
  contents: read

jobs:
  deploy:
    uses: fmidev/.github/.github/workflows/openshift-deploy.yaml@0.4.0
    with:
      APPLICATION_NAME: 'my-app'
      DEPLOY_CONFIG_REPOSITORY_BASE_PATH: 'pak'
      DEPLOY_CONFIG_REPOSITORY_ENVIRONMENTS: 'development,staging'
    secrets: inherit
```

## Promoting an existing version

Leave production out of the tag-push deployment above and promote it separately
with `openshift-promote.yaml`. It reuses the image and the Helm chart that were
published when the version was built, so a promotion cannot change what was
tested in staging:

```yaml
on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Released version to deploy, e.g. 1.2.3'
        required: true
        type: string

jobs:
  deploy:
    uses: fmidev/.github/.github/workflows/openshift-promote.yaml@0.4.0
    with:
      APPLICATION_NAME: 'my-app'
      APPLICATION_VERSION: ${{ inputs.version }}
      DEPLOY_CONFIG_REPOSITORY_BASE_PATH: 'pak'
      DEPLOY_CONFIG_REPOSITORY_ENVIRONMENTS: 'production'
      SLACK_MESSAGE: 'My App version upgrade ${{ inputs.version }} to production'
    secrets: inherit
```

The promotion runs in three stages:

1. **Verify.** The version is checked against the SemVer format, and the
   container image and the Helm chart for it must exist in the registry. The
   image is looked up first. A version that was never built fails here, before
   anything in GitOps has changed.
2. **Update.** `update-openshift-application-version.yaml` rewrites the version
   references for every environment in
   `DEPLOY_CONFIG_REPOSITORY_ENVIRONMENTS`.
3. **Announce.** When the `SLACK_WEBHOOK_URL` secret is available, the
   promotion is posted to Slack through
   `post-to-slack-deploy-automation-notifier.yaml`. With no webhook the stage is
   skipped and the promotion still succeeds, so the secret is optional.

`SLACK_MESSAGE` sets the text. It defaults to
`<APPLICATION_NAME> version upgrade <APPLICATION_VERSION> to <DEPLOY_CONFIG_REPOSITORY_ENVIRONMENTS>`,
which is worth overriding when the application is known by a different name than
its repository.

To require an approval before the promotion runs, put a job with an
`environment:` in front of it in the calling workflow and configure required
reviewers for that environment. A job that calls a reusable workflow cannot
declare an `environment:` of its own.

## What the GitOps repository must look like

`update-openshift-application-version.yaml` is the last mile of both deployment
paths, and it expects, for every environment it is given:

```
<DEPLOY_CONFIG_REPOSITORY_BASE_PATH>/<APPLICATION_NAME>/<environment>/Chart.yaml
<DEPLOY_CONFIG_REPOSITORY_BASE_PATH>/<APPLICATION_NAME>/<environment>/values.yaml
```

It rewrites `.version` and `.dependencies[0].version` in `Chart.yaml`, and
`.<APPLICATION_NAME>-chart.application.deployment.image.tag` in `values.yaml`,
then commits and pushes. A missing directory fails the job with a message naming
the path, rather than at the first `yq` call.

Note that `APPLICATION_NAME` is the directory name in GitOps, not necessarily the
repository name: `cdn.fmi.fi` deploys as `cdn-fmi-fi`.

Several applications share one GitOps repository, so two deployments running at
once race on the push. A rejected push is rebased and retried rather than
failing the deployment. Concurrency groups cannot solve this — they are scoped to
the repository that calls the workflow, and the racing runs are in different
repositories.

## Upgrading a caller from 0.3.x to 0.4.0

Three changes are breaking, and each fails loudly and immediately rather than
part-way through a deployment.

**Workflow files are now `.yaml`, not `.yml`.** Update the path together with the
tag:

```diff
-    uses: fmidev/.github/.github/workflows/openshift-deploy.yml@0.3.3
+    uses: fmidev/.github/.github/workflows/openshift-deploy.yaml@0.4.0
```

A caller that bumps the tag without renaming gets "workflow was not found" and
nothing runs. Callers pinned to `@0.3.3` are unaffected and keep working.

**`openshift-deploy.yaml` no longer declares `HELM_REPOSITORY_PRIVATE_DEPLOY_KEY`.**
It was required but never used. Callers using `secrets: inherit` need no change.
A caller passing secrets explicitly must drop that line, or the call fails with
"invalid input".

**`deploy-via-rsync.yaml` and `deploy-to-fmi-openshift.yaml` are gone.** Both were
obsolete and nothing in the organisation called either. `deploy-via-rsync` had in
any case never worked: `ssh-add` ran in a step where the deploy key was not in
the environment, so the key file it wrote was empty. `deploy-to-fmi-openshift`
was the original single-job pipeline, superseded by `openshift-deploy.yaml`
together with `openshift-promote.yaml`. Callers pinned to `@0.3.3` keep working;
there is no replacement for either at `0.4.0`.

Everything else is additive: `IMAGE_TAG_PREFIX`, `IMAGE_TAG_SUFFIX`,
`IMAGE_PLATFORMS`, `IMAGE_FILE`, `HELM_CHART_PATH`, `HELM_VERSION`,
`HELM_CHART_NAME` and the `IMAGE_BUILD_ARGS` secret all default to the previous
behaviour.

## Releasing a new version of this repository

Sub-workflow references inside this repository are pinned to a tag as well, so a
release is two steps:

1. Update the `@<version>` references in `openshift-deploy.yaml` and
   `openshift-promote.yaml` to the version about to be released.
2. Tag the merge commit with that version.
