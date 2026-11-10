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
`uses: fmidev/.github/.github/workflows/openshift-deploy.yaml@0.5.1`.

## Checks on a change here

Nothing in this repository is built or tested by the repository itself. Every
workflow runs on behalf of some other repository, so a mistake lands on whoever
called it, part-way through a deployment. `lint.yaml` is the only gate before
that, and all three of its jobs must stay green:

- **actionlint** for expressions that do not parse, unknown contexts, actions on
  retired runners, and shellcheck findings in `run:` scripts. The
  `ubuntu-26.04` label is declared in `.github/actionlint.yaml` until an
  actionlint release knows it.
- **yamllint** `--strict`, configured in `.yamllint.yaml`, for the defects that
  change meaning: a duplicate key silently discarding a value, a tab, broken
  indentation.
- **zizmor** for the failure modes specific to shared CI: an untrusted value
  reaching a shell, a mutable action reference, a credential left behind in the
  workspace. Accepted findings are listed with their reasons in
  `.github/zizmor.yml`; the audit is expected to report nothing at all.

All three run in containers pinned by digest, so a lint result is reproducible.
Dependabot keeps the actions current but cannot see a container named inside a
`run:` step, so bump those digests along with the tag.

## Pinning

Every external action is pinned by SHA, with the version in a trailing comment
so it stays readable and Dependabot can bump both together:

```yaml
uses: docker/login-action@81c6f93d4bc991e510b342a00b419b3bb4efc5c6 # v4.7.0
```

Reusable workflows from this repository stay on their release tag, so that one
tag describes the whole set. That exception is encoded as a policy in
`.github/zizmor.yml` rather than left to reviewer memory.

Runners are pinned to `ubuntu-26.04` rather than left on `ubuntu-latest`, so that
a GitHub-side image migration cannot change what repositories deploy with,
without a commit here. `ubuntu-26.04` has been generally available since
September 2026, and `ubuntu-latest` is moving to it between 19 October and
19 November 2026.

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
    uses: fmidev/.github/.github/workflows/openshift-deploy.yaml@0.5.1
    with:
      APPLICATION_NAME: 'my-app'
      DEPLOY_CONFIG_REPOSITORY_BASE_PATH: 'tenant-id'
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
      description:
        description: 'What changes, e.g. Adds the new radar layer'
        required: false
        type: string
      jira:
        description: 'Jira issue, e.g. PAK-1234'
        required: false
        type: string
      issue:
        description: 'GitHub issue: 123, #123 or the issue URL'
        required: false
        type: string

jobs:
  deploy:
    uses: fmidev/.github/.github/workflows/openshift-promote.yaml@0.5.1
    with:
      APPLICATION_NAME: 'my-app'
      APPLICATION_DISPLAY_NAME: 'My App'
      APPLICATION_VERSION: ${{ inputs.version }}
      DEPLOY_CONFIG_REPOSITORY_BASE_PATH: 'tenant-id'
      DEPLOY_CONFIG_REPOSITORY_ENVIRONMENTS: 'production'
      CHANGE_DESCRIPTION: ${{ inputs.description }}
      JIRA_ISSUE: ${{ inputs.jira }}
      GITHUB_ISSUE: ${{ inputs.issue }}
    secrets: inherit
```

The promotion runs in four stages:

1. **Compose.** The Slack message is composed and its inputs checked, as
   described below, so a mistyped issue fails before anything is looked up.
2. **Verify.** The version is checked against the SemVer format, and the
   container image and the Helm chart for it must exist in the registry. The
   image is looked up first. A version that was never built fails here, before
   anything in GitOps has changed.
3. **Update.** `update-openshift-application-version.yaml` rewrites the version
   references for every environment in
   `DEPLOY_CONFIG_REPOSITORY_ENVIRONMENTS`.
4. **Announce.** When the `SLACK_WEBHOOK_URL` secret is available, the
   promotion is posted to Slack through
   `post-to-slack-deploy-automation-notifier.yaml`. With no webhook the stage is
   skipped and the promotion still succeeds, so the secret is optional.

To require an approval before the promotion runs, put a job with an
`environment:` in front of it in the calling workflow and configure required
reviewers for that environment. A job that calls a reusable workflow cannot
declare an `environment:` of its own.

### The Slack message

The change management guide asks for a change announced on #fmi-muutokset to
say who makes it and what changes, and to link the Jira issue when there is
one. The message is therefore a single line naming the version, the person who
started the run and the description, followed by the links:

```
My App 1.2.3 to production by mmeikala: Adds the new radar layer (https://jira.fmi.fi/browse/PAK-123, https://github.com/fmidev/my-app/issues/45, https://github.com/fmidev/my-app/releases/tag/1.2.3)
```

The links are bare URLs, which Slack links by itself. The deploy automation
notifier is a Slack Workflow Builder trigger, and Workflow Builder shows the
text it is given as it is: link markup such as `<url|text>` would appear
literally, and so would a mention such as `<!channel>`.

- The name is `APPLICATION_DISPLAY_NAME`, or `APPLICATION_NAME` when not set.
  Line breaks in it become spaces.
- The person is the GitHub login of whoever started the run; re-running
  failed jobs keeps them.
- `CHANGE_DESCRIPTION` follows a colon, with line breaks turned into spaces.
  Without it there is no colon.
- In the parentheses, in this order:
  - `JIRA_ISSUE`, e.g. `PAK-123`, under `JIRA_BASE_URL`, which defaults to
    `https://jira.fmi.fi`, also when a caller passes it empty; it must be a
    plain http(s) URL, without `?` or `#`. Left out when not given.
  - `GITHUB_ISSUE`, an issue number in the calling repository (`45` or `#45`),
    or the URL of an issue on this instance. Left out when not given.
  - The release page of the version, always. GitHub shows it for a tag
    without a release too. It assumes that the release is tagged in the
    calling repository, with the bare version (`1.2.3`, not `v1.2.3`), as
    `openshift-deploy.yaml` expects.

Both issues are typically typed into a form, so surrounding whitespace, a
lowercase Jira key, and a trailing `/` or `#issuecomment-…` on an issue URL are
accepted. Anything else that is not a Jira key or a GitHub issue fails the
promotion before anything has been looked up or changed.

GitHub drops a job output that contains the value of a secret. The message is
composed in a job that holds none, but should it be dropped anyway, a shorter
`<name> <version> to <environments> by <login> (<release URL>)` is posted
instead of an empty message.

`SLACK_MESSAGE` is the earlier interface: when set, it is posted verbatim
instead. The inputs above are then neither used nor validated; when any of
them is given, a warning says they are unused. Use `APPLICATION_DISPLAY_NAME`
instead.

## What the GitOps repository must look like

`update-openshift-application-version.yaml` is the last mile of both deployment
paths, and it expects, for every environment it is given:

```
<DEPLOY_CONFIG_REPOSITORY_BASE_PATH>/<APPLICATION_NAME>/<environment>/Chart.yaml
<DEPLOY_CONFIG_REPOSITORY_BASE_PATH>/<APPLICATION_NAME>/<environment>/values.yaml
```

It rewrites `.version` and `.dependencies[0].version` in `Chart.yaml`, and
`.<APPLICATION_NAME>-chart.application.deployment.image.tag` in `values.yaml`
(override the key with `HELM_CHART_KEY` when it is not named after the
application), then commits and pushes. Every rewritten path must already exist:
`yq` would otherwise silently create it, producing a green deployment that
deployed nothing. A missing directory fails the job with a message naming
the path, rather than at the first `yq` call.

Note that `APPLICATION_NAME` is the directory name in GitOps, not necessarily the
repository name: `cdn.fmi.fi` deploys as `cdn-fmi-fi`.

Several applications share one GitOps repository, so two deployments running at
once race on the push. A rejected push is rebased and retried rather than
failing the deployment. Concurrency groups cannot solve this — they are scoped to
the repository that calls the workflow, and the racing runs are in different
repositories.

## Upgrading a caller

Callers pin a tag of this repository, so nothing changes under them until they
bump it. [`CHANGELOG.md`](CHANGELOG.md) says what each release changed and which
changes require a change in the calling repository.

Note that 0.4.0 renamed every workflow file from `.yml` to `.yaml`, so a caller
moving to it has to update the path as well as the tag.

## Releasing a new version of this repository

Sub-workflow references inside this repository are pinned to a tag as well, so a
release is three steps:

1. Move the `CHANGELOG.md` entry from unreleased to the version and date being
   released, calling out anything a caller has to change.
2. Update the `@<version>` references in `openshift-deploy.yaml` and
   `openshift-promote.yaml` to that version.
3. Tag the merge commit with it.

Steps 2 and 3 are not atomic on GitHub: between the merge and the tag, a caller
already on the new version has no workflows to resolve. Tag promptly.
