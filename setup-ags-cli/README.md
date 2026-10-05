# Setup AGS CLI

Installs and caches [AGS CLI](https://github.com/AccelByte/accelbyte-ags-cli) on a GitHub Actions runner, and optionally logs it in. Afterwards `ags` is on `PATH` and configured, so your own steps are plain CLI calls with no environment plumbing.

```yaml
- uses: AccelByte/setup-ags-cli@v1
  with:
    version: '0.5.1'
    base-url: ${{ vars.AGS_BASE_URL }}
    client-id: ${{ secrets.AGS_CLIENT_ID }}
    client-secret: ${{ secrets.AGS_CLIENT_SECRET }}

- run: ags doctor   # ...or any other ags command
```

For a complete deployment pipeline built on this action, see [extend-github-actions-template](https://github.com/AccelByte/extend-github-actions-template).

## Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `version` | no | `latest` | AGS CLI version, e.g. `0.5.1`. A leading `v` is accepted. `latest` resolves the newest release at run time. |
| `base-url` | to authenticate | — | AGS environment base URL, e.g. `https://dev.yourstudio.accelbyte.io`. Exported as `AGS_BASE_URL`. |
| `client-id` | to authenticate | — | Confidential IAM client ID. Pass from a secret, never a literal. |
| `client-secret` | to authenticate | — | Confidential IAM client secret. Pass from a secret, never a literal. |
| `github-token` | no | `${{ github.token }}` | Used *only* to resolve `version: latest` through the GitHub API. The default is enough. |

## Outputs

| Output | Description |
| --- | --- |
| `version` | The concrete version installed, after `latest` is resolved. Use it to pin a later build to the same CLI, or to print it in a job summary. |
| `cache-hit` | `true` when the CLI was restored from cache rather than downloaded. |

## Install-only mode

Omit both credentials and the action installs the CLI without logging in. Useful for anything that doesn't need a session:

```yaml
- uses: AccelByte/setup-ags-cli@v1
- run: ags --version
```

Passing exactly one of `client-id` / `client-secret` is treated as a mistake and fails with a named error — that combination is almost always a typo'd secret name, and failing loudly beats a confusing `401` three steps later.

## Requirements

- **Linux or macOS runners.** The upstream installer is a POSIX shell script; the action fails fast with a clear message on Windows rather than letting `curl` pipe into a shell that isn't there.
- **Docker**, if you're going on to run `ags extend image-upload`. Present on GitHub-hosted Ubuntu runners.

---

The rest of this document explains *why* the action does what it does. You don't need it to use the action, but it's worth reading before you work around something.

## How the CLI gets installed

Two steps handle this: cache, then install on a miss.

### Cache

```yaml
- uses: actions/cache@v4
  with:
    path: ~/.ags-cli
    key: ags-${{ steps.resolve.outputs.version }}-${{ runner.os }}-${{ runner.arch }}
```

Every workflow run starts on a clean runner, so without a cache the CLI archive is downloaded from GitHub Releases on every single run. Caching it means that download happens only on the first run and after a version bump — so a slow or unavailable GitHub Releases endpoint won't fail an otherwise-cached job.

The cache key is what makes version upgrades work without any manual cache clearing:

- **The resolved version** — a cache entry is immutable once written; GitHub will not overwrite an existing key. Putting the version in the key means bumping `version` produces a new key, which misses, which triggers a fresh install. Leave the version out and you'd be pinned to whatever binary was cached first, forever.
- **`runner.os` / `runner.arch`** — the cached artifact is a compiled binary. A Linux x86_64 build is not valid on any other target. If you add a matrix or move to a different runner image, the key changes with it.

Note the ordering constraint: the version has to be resolved *before* the cache step, because the key contains it. That's why `version: latest` costs one GitHub API call even on a cache hit. Pin `version` and the call is skipped entirely.

### Install

```yaml
- if: steps.cache.outputs.cache-hit != 'true'
  run: |
    set -euo pipefail
    export CARGO_HOME="$HOME/.ags-cli"
    mkdir -p "$CARGO_HOME"
    curl --proto '=https' --tlsv1.2 -LsSf \
      "https://github.com/AccelByte/accelbyte-ags-cli/releases/download/v${AGS_VERSION}/accelbyte-ags-cli-installer.sh" \
      | sh
```

This runs only on a cache miss — a first run, a version bump, or an expired cache. On a hit it's skipped entirely.

- **`CARGO_HOME`** — the installer is a `cargo-dist` shell installer, which installs into `$CARGO_HOME/bin`. Overriding it redirects the binary into `~/.ags-cli/bin` instead of the default `~/.cargo/bin`, so it lands inside the directory the cache step covers. The two paths have to agree or the cache does nothing.
- **`--proto '=https' --tlsv1.2`** — refuse any protocol downgrade and set a TLS floor. Standard hygiene for a `curl | sh` install.
- **`-f`** — fail on an HTTP error response. Without it, a 404 body gets piped straight into `sh`, which is both useless and unsafe.
- **`-L`** — follow redirects. GitHub release assets redirect to a CDN.

## Runner state and authentication

Four environment variables make the CLI behave on a clean runner. The action writes them to `$GITHUB_ENV`, so they apply to every step after it.

| Variable | Why |
| --- | --- |
| `AGS_NO_KEYCHAIN=1` | Hosted runners have no OS keychain — no macOS Keychain, no Windows Credential Manager, no running Linux Secret Service. Setting this goes straight to file-based token storage instead of attempting a keychain call that will fail. |
| `AGS_PROFILE=default` | Resolved before the CLI's first-run profile setup, so it works on a clean runner regardless of what is on disk. Without it the CLI fails with `No active profile`. |
| `AGS_HOME=$RUNNER_TEMP/ags-home` | Puts CLI state — including the access token — in a temp directory **outside the repository**. |
| `AGS_BASE_URL` | Set from the `base-url` input, so later `ags` calls don't each need it. Only written if the input was given. |

> **Why `AGS_HOME` matters more than it looks.** If you go on to build a container image with the repository root as the Docker build context, any CLI state written inside the repository would be visible to the build and could end up baked into an image layer. Pointing `AGS_HOME` at `$RUNNER_TEMP` keeps the token out of the build context entirely. Don't move it into the workspace.

Authentication is the standard client-credentials flow:

```yaml
ags auth login --grant client-credentials --no-input
```

`--no-input` makes the CLI fail rather than prompt if anything is missing.

### What the action does *not* export

Credentials never reach `$GITHUB_ENV`. The login leaves a token in `AGS_HOME` and later steps read that, so `client-secret` does not become job-wide environment that every subsequent step and action can see.

The action also masks `client-secret` with `::add-mask::`. Values that came from `secrets.*` are masked by GitHub already, but a caller passing a literal, or a value read from a file, gets no masking for free.

## On pinning

`version` defaults to `latest`, which is the right default for a shared action — a hardcoded version in a published action silently rots, and every caller inherits the rot.

For your own pipelines, prefer pinning:

```yaml
    version: '0.5.1'
```

Upgrading is then a deliberate one-line edit, not something that lands in your pipeline unannounced. You can capture what a green run actually used from the `version` output and pin to that.

## Troubleshooting

| Symptom | Cause and fix |
| --- | --- |
| `No active profile` | You're running `ags` before this action's step, so `AGS_PROFILE` / `AGS_HOME` aren't set yet. Move the call after it. |
| `Could not resolve the latest AGS CLI release` | The GitHub API call failed or was rate-limited. Pin `version` to a known release. |
| `404` during install | The `version` input doesn't match a published release tag. Check the [releases page](https://github.com/AccelByte/accelbyte-ags-cli/releases). |
| `setup-ags-cli supports Linux and macOS runners` | You're on a Windows runner. Use `ubuntu-latest` or a macOS runner. |
| Version bump doesn't take effect | The cache key includes the resolved version, so this normally resolves itself. If the key was edited, clear the entry under **Actions → Caches**. |
| Auth fails at login | Confirm the IAM client is **Confidential**, not Public, and that `base-url` points at the right environment. |

Run `ags doctor` against the same base URL and client to check config, auth, and connectivity independently of CI.

## Security notes

- `client-secret` is a long-lived credential. Store it in repository or organization **secrets**, never in variables — variables are not masked in logs. Rotate it periodically and on personnel changes.
- Grant the IAM client only the permissions your pipeline actually needs, scoped to one namespace. If the secret is compromised, the blast radius is whatever you granted it.
- Keep `AGS_HOME` outside the workspace — see [Runner state and authentication](#runner-state-and-authentication) for why.
