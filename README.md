# Deploy Extend App

A GitHub Actions workflow that builds your Extend app's container image, pushes it to the app's registry, and deploys it — on every push to `main`, or on demand.

It runs [AGS CLI](https://github.com/AccelByte/accelbyte-ags-cli) on a standard GitHub-hosted runner. The workflow is a plain YAML file you copy into your own repository and edit.

Installing and authenticating the CLI is delegated to [`AccelByte/setup-ags-cli`](https://github.com/AccelByte/setup-ags-cli), because those steps carry runner-specific workarounds that are ours to maintain rather than yours. Everything else — when to deploy, what to build, how to report the result — lives in this file, in the open, for you to change.

```
push to main  →  build image  →  push to app registry  →  deploy  →  wait for rollout
```

Workflow file: `.github/workflows/deploy-extend-app.yml`

> **New to GitHub Actions?** A *workflow* is this YAML file. GitHub reads it automatically and runs it on the events listed at the top — here, a push to `main` or a manual trigger. Each run executes on a *runner*: a fresh, temporary Linux machine GitHub provides, so nothing is installed on your own computer and every run starts from a clean slate. A workflow is made of *jobs*, and each job is a list of *steps* that run top to bottom. You don't install or host anything — commit the file and GitHub takes it from there.

## Prerequisites

- **An Extend app that already exists.** This workflow deploys to an app, it does not create one. Create the app first in the Admin Portal or with AGS CLI.
- **A `Dockerfile` in your repository root.** The build context is the repository root (`--work-dir .`).
- **An IAM client.** [Create an IAM client](https://docs.accelbyte.io/gaming-services/modules/foundations/identity-access/authorization/manage-access-control-for-applications/) with client type `confidential` and assign the required permissions listed below. Keep a copy of the `Client ID` and `Client Secret`.

    - For AGS Private Cloud customers:
        - `ADMIN:NAMESPACE:{namespace}:EXTEND:REPOCREDENTIALS` [READ]
        - `ADMIN:NAMESPACE:{namespace}:EXTEND:APP` [READ]
        - `ADMIN:NAMESPACE:{namespace}:EXTEND:DEPLOYMENT` [CREATE]
    - For AGS Public Cloud customers:
        - Extend > Extend app image repository access (Read)
        - Extend > App Management (Read)
        - Extend > Deployment Management (Create)

  These are the minimum permissions the workflow needs: read the app, mint registry credentials, create a deployment. Don't widen them by copying a permission set from other Extend tooling — this credential lives in a repository indefinitely, and the list above deliberately cannot create, update, or delete the app.

## Setup

### 1. Add the workflow file to your repository

The workflow must live at `.github/workflows/deploy-extend-app.yml` on your repository's default branch. If you started from the sample app, it's already there — skip to the next step. If you're adding deployment to your own repo, copy that file into the same path and commit it. GitHub only picks up a workflow once the file is committed to the branch, so it won't appear under the **Actions** tab until then.

### 2. Add repository variables

In your GitHub repository, go to **Settings → Secrets and variables → Actions → Variables**.

| Variable | Description | Example |
| --- | --- | --- |
| `AGS_BASE_URL` | Your AGS environment base URL | `https://dev.yourstudio.accelbyte.io` |
| `AGS_NAMESPACE` | The namespace the app lives in | `yourgame` |
| `EXTEND_APP_NAME` | The name of the existing Extend app | `guild-service` |

### 3. Add repository secrets

In your GitHub repository, go to **Settings → Secrets and variables → Actions → Secrets**.

| Secret | Description |
| --- | --- |
| `AGS_CLIENT_ID` | Confidential client ID |
| `AGS_CLIENT_SECRET` | Confidential client secret |

The first step of the workflow checks all five and fails with a specific message naming anything that is missing, so a misconfigured repository fails in about ten seconds rather than part-way through a deploy.

### 4. Push to `main`

That's the whole setup. The workflow runs on every push to the **`main`** branch (and on manual runs — see [Manual deploy and rollback](#manual-deploy-and-rollback)). If your repository's default branch has a different name, either rename it to `main` or change `branches: [main]` at the top of the workflow to match — otherwise nothing will trigger.

## Configuration

Tunable values live in the `env:` block at the top of the workflow.

| Variable | Default | Description |
| --- | --- | --- |
| `AGS_VERSION` | `0.5.1` | AGS CLI version to install. Passed to the setup action, which uses it in its cache key. Set to `latest` to always take the newest release. |
| `WAIT_LIMIT` | `600` | Seconds to wait for the rollout before giving up. |
| `WAIT_INTERVAL` | `10` | Seconds between rollout status polls. |
| `IMAGE_TAG` | commit SHA | Tag applied to the built image. |

> **On pinning.** `AGS_VERSION` fixes the CLI rather than tracking the latest release, so upgrading is a deliberate one-line edit instead of something that lands in your pipeline unannounced — and re-running an old commit installs the CLI that commit was tested with. Set it to `latest` if you would rather have the newest release and don't need reproducible builds.

### Keeping the pinned version current

A pin only stays useful if someone bumps it. Dependabot won't: it updates `uses:` references, not the value of an arbitrary environment variable, so `AGS_VERSION` is invisible to it and will quietly go stale.

[Renovate](https://docs.renovatebot.com/) can track it, with a custom manager in your `renovate.json`:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended"],
  "customManagers": [
    {
      "customType": "regex",
      "managerFilePatterns": [".github/workflows/deploy-extend-app.yml"],
      "matchStrings": ["AGS_VERSION:\\s*['\"]?(?<currentValue>\\d+\\.\\d+\\.\\d+)['\"]?"],
      "depNameTemplate": "AccelByte/accelbyte-ags-cli",
      "datasourceTemplate": "github-releases",
      "versioningTemplate": "semver",
      "extractVersionTemplate": "^v(?<version>.+)$"
    }
  ]
}
```

You'll get a pull request per AGS CLI release, with the changelog attached — so the upgrade stays a deliberate decision, it just arrives on your doorstep instead of waiting to be remembered.

Three details in there are load-bearing:

- **`extractVersionTemplate`** strips the leading `v`. Releases are tagged `v0.5.1` but the env var holds `0.5.1`; without this Renovate would write the tag verbatim and the installer URL would end up looking for `vv0.5.1`.
- **The `\\d+\\.\\d+\\.\\d+` requirement** means `AGS_VERSION: 'latest'` isn't matched at all. If you switch to tracking latest, Renovate ignores the line instead of trying to "upgrade" the word `latest`. The quotes are optional in the pattern, so an unquoted `AGS_VERSION: 0.5.1` is still tracked.
- **`managerFilePatterns`** is the current field name — older Renovate configs call it `fileMatch`, which still works but is deprecated. A bare string is treated as a glob; wrap it in `/.../` if you want a regex.

The `uses: AccelByte/setup-ags-cli@v1` reference needs no custom rule. `@v1` is a floating major tag, so bug fixes arrive without any version change at all. If you'd rather pin it exactly — `@v1.2.3` — Renovate's and Dependabot's built-in GitHub Actions managers both handle that automatically.

### Concurrency and timeout

Two job-level safety settings sit outside the `env:` block:

- **`concurrency`** serializes deploys per branch, with `cancel-in-progress: false`. If a second push lands while a deploy is still running, it waits for the first to finish rather than racing it — two rollouts to the same app can't overlap, and an in-flight rollout is never cancelled part-way through (which could leave the app half-updated).
- **`timeout-minutes: 30`** caps the whole job. `WAIT_LIMIT` only bounds the rollout step; this covers the case where the image build itself hangs, so a stuck run can't run up to GitHub's 6-hour default before it's killed.

## AGS CLI setup

One step installs the CLI, caches it, and logs in:

```yaml
      - name: Setup AGS CLI
        id: setup
        uses: AccelByte/setup-ags-cli@v1
        with:
          version: ${{ env.AGS_VERSION }}
          base-url: ${{ vars.AGS_BASE_URL }}
          client-id: ${{ secrets.AGS_CLIENT_ID }}
          client-secret: ${{ secrets.AGS_CLIENT_SECRET }}
```

Afterwards `ags` is on `PATH` and `AGS_BASE_URL`, `AGS_HOME`, `AGS_PROFILE` and `AGS_NO_KEYCHAIN` are set for every later step — which is why the commands further down are plain CLI calls with no environment blocks of their own.

Those four variables exist to work around things that are true of GitHub-hosted runners: no OS keychain, no CLI state directory, and a token that has to stay out of the Docker build context. The action owns them so that a fix reaches you by bumping a tag rather than by you editing this file.

**If you want the detail** — how the cache key is built, why `CARGO_HOME` is overridden, why the token must not live in the workspace, what the action deliberately does *not* export — it's all in [the action's README](https://github.com/AccelByte/setup-ags-cli#how-the-cli-gets-installed). You don't need it to use this workflow, but read it before you work around anything.

## Build and push

```yaml
- name: Build and push image
  run: |
    ags extend image-upload \
      --namespace "$AGS_NAMESPACE" \
      --app "$EXTEND_APP" \
      --image-tag "$IMAGE_TAG" \
      --work-dir . \
      --dockerfile Dockerfile \
      --platform linux/amd64 \
      --login \
      --retry-limit 2
```

One command builds the image and pushes it to the app's own registry. `--login` handles registry authentication using the session from the previous step, so there is no second set of registry credentials to manage.

- **`--platform linux/amd64`** is deliberate. If you build the same image on an Apple Silicon machine without this flag you get an `arm64` image, which starts fine locally and then fails on the cluster with `exec format error`. Pinning the platform in CI means the image you deploy is the image the cluster can run.
- **`--image-tag` defaults to the commit SHA**, which gives every deploy an immutable, traceable tag. This is also what makes rollback trivial (below). Avoid `latest` here; it destroys both the audit trail and the rollback path.

## Deploy and wait

```yaml
- name: Deploy and wait for rollout
  run: |
    ags extend deploy-app \
      --namespace "$AGS_NAMESPACE" \
      --app "$EXTEND_APP" \
      --json "$(printf '{"imageTag":"%s"}' "$IMAGE_TAG")" \
      --wait --wait-limit "$WAIT_LIMIT" --wait-interval "$WAIT_INTERVAL" \
      --api-scope admin --api-version v5 \
      --format json --no-input --yes
```

`deploy-app` takes the image tag in a JSON request body (`--json`) rather than as a flag.

`--wait` polls until the rollout reaches a terminal state or the wait limit expires. The step captures the exit code with `|| code=$?` and re-raises it on the last line, so the run still goes red on a failed deploy; the `Summary` step then maps that code to a human-readable result:

| Exit code | Result | Meaning |
| --- | --- | --- |
| `0` | deployed | Rollout completed and the app is healthy. |
| `3` | failed | Rollout reached a terminal failure. Check the app logs before retrying. |
| `6` | timed out | `WAIT_LIMIT` elapsed with the rollout still in progress. |
| other | failed | Something else went wrong — a bad request, an auth failure, a network error. |

> **Exit `6` is not a rollback.** A timeout means CI stopped watching, not that the deployment stopped. The rollout may still succeed a minute later. Check the app's actual status before redeploying, or you risk stacking a second deployment on top of one that was about to finish. If your app is legitimately slow to start, raise `WAIT_LIMIT` rather than treating timeouts as normal.

Either way, the final step writes a job summary — app, namespace, image tag, result, and a hint for codes `3` and `6` — so the outcome is visible on the run page without opening the logs.

## Adding tests

There's a commented placeholder between configuration and install:

```yaml
- name: Test
  run: make test
```

Anything that exits non-zero there stops the workflow before any image is built or pushed. Uncomment it and point it at your test command.

## Manual deploy and rollback

The workflow also has a `workflow_dispatch` trigger. Under **Actions → Deploy Extend App → Run workflow** you can choose a ref and optionally override the image tag.

**To roll back:** select the last known-good commit or tag as the ref and leave the image tag input empty. The workflow checks out that ref, rebuilds it, tags it with that commit's SHA, and deploys it.

> **Watch the image-tag override.** The tag input does not choose *what* to build — it only labels whatever the selected ref contains. Running against `main` with an older SHA typed into the tag field will build current `main` and push it under a misleading tag. Use the ref selector for rollback; use the tag input only when you want a non-SHA label such as a release tag.

## Troubleshooting

| Symptom | Cause and fix |
| --- | --- |
| `Missing repository variable …` or `Missing secret …` | The config check found something unset. The message names which one and where to set it. |
| `No active profile` | An `ags` command ran before the `Setup AGS CLI` step, so `AGS_PROFILE` and `AGS_HOME` weren't set yet. Move it after that step. |
| Image builds, app never becomes healthy | Usually a crash at startup — a missing environment variable or config the app needs at boot. Check the app logs in the Admin Portal. |
| `exec format error` in the app logs | Architecture mismatch. Confirm `--platform linux/amd64` is still present and that your base image has an `amd64` variant. |
| Auth fails in the `Setup AGS CLI` step | Confirm the IAM client is **Confidential**, not Public, and that `AGS_BASE_URL` points at the right environment. |
| CLI version bump doesn't take effect | Change `version` on the `Setup AGS CLI` step, not an `env:` var. The cache key includes it, so the bump takes effect on the next run. |
| Deploy rejected as unauthorized | The IAM client is missing a permission. Compare it against the list in the prerequisites — the error response names the permission string it expected. |

Run `ags doctor` locally against the same base URL and client to check config, auth, and connectivity independently of CI.

## Security notes

- `AGS_CLIENT_SECRET` is a long-lived credential in a repository. Rotate it periodically and on personnel changes.
- Grant only the permissions listed in the prerequisites, scoped to one namespace. If the secret is compromised, the blast radius is whatever you granted it — and that list deliberately cannot delete the app.
- `AGS_HOME` is kept outside the workspace by `setup-ags-cli`, so the access token can't be baked into an image layer — see [Runner state and authentication](https://github.com/AccelByte/setup-ags-cli#runner-state-and-authentication) for why that matters.
- Never move credentials from `secrets` into `vars`. Repository variables are not masked in logs.
- The `permissions: contents: read` block at the job level is intentional — this workflow never needs write access to the repository.