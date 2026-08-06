# release-config

Shared release pipeline for Codam's containerised Node services: build a Docker image,
let [semantic-release](https://semantic-release.gitbook.io/) decide the version from the
commit history, then tag, push to [GHCR](https://ghcr.io) and cut a GitHub release.

Both the workflow **and** the semantic-release config live in this one repo. The config is
written at runtime by the workflow, so there is a single source of truth and no way for the
two to drift apart. Consumer repos hold no release config at all.

## Usage

Create a `.github/workflows/release.yml` workflow in a consumer repo with:

```yaml
name: Build, Publish & Release

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  deploy:
    uses: codam-coding-college/release-config/.github/workflows/release.yml@main
```

Do not create your own release config in the consumer repo. The workflow writes a
`release.config.cjs` at runtime, and a `release.config.js` left behind would take precedence
over it.
It deliberately sets no `repositoryUrl`, so semantic-release infers it from the checkout's git
remote, and the image path reaches `publishCmd` through the `$IMAGE_NAME` environment variable
rather than the config.

Remove `semantic-release` and every `@semantic-release/*` package from the consumer's
`devDependencies` as well. The workflow installs its own pinned toolchain, so consumer repos
declare nothing release-related at all.

### Running tests first

```yaml
jobs:
  tests:
    uses: ./.github/workflows/test-run.yml

  deploy:
    needs: [tests]
    uses: codam-coding-college/release-config/.github/workflows/release.yml@main
```

## Inputs

All optional.

| Input | Default | Notes |
|---|---|---|
| `image-name` | calling repo's slug | Path under `ghcr.io`. See the caveat below. |
| `node-version` | `lts/*` | Node version used to run semantic-release, not to build the image. |
| `submodules` | `recursive` | |
| `dockerfile` | `Dockerfile` | |
| `context` | `.` | |

There is no `branch` input: semantic-release's default branch list already covers both
`main` and `master`. Control *when* the workflow runs via the caller's `on:` trigger.

### `image-name` caveat

`GITHUB_TOKEN` can only push to GHCR packages owned by the calling repo's org, so this
input cannot redirect an image into a different organisation. It is only useful when the
image path differs from the repo slug *within* the same org. If a repo has moved org,
update whatever pulls the image rather than overriding this.

## Secrets

- `GITHUB_TOKEN`: used by default. Needs no passing; every job gets one.
- `RELEASE_TOKEN`: optional, and preferred over `GITHUB_TOKEN` when present. Pass it
  explicitly to opt in:

  ```yaml
    secrets:
      RELEASE_TOKEN: ${{ secrets.RELEASE_TOKEN }}
  ```

  Only needed when the release must trigger other workflows, which `GITHUB_TOKEN`
  deliberately cannot do, or must push past branch protection it cannot bypass.
  Note that `secrets: inherit` passes it implicitly — avoid it unless you want that.

## What it publishes

For version `X.Y.Z` on the default branch:

- `ghcr.io/<image-name>:X.Y.Z`
- `ghcr.io/<image-name>:latest`
- git tag `vX.Y.Z` plus a `chore(release): X.Y.Z [skip ci]` commit updating `package.json`
  and `package-lock.json` (if present)
- a GitHub release with generated notes

## Release toolchain

`semantic-release` and its plugins are pinned in the `SEMANTIC_RELEASE_PACKAGES` env var in
[`release.yml`](.github/workflows/release.yml). Bumping a version there updates
every consumer repo on its next run - that is the only place these versions are declared.

They are installed into `$RUNNER_TEMP`, outside the checkout, for two reasons:

1. Installing into the repo would rewrite `package-lock.json`, and that file is a
   `@semantic-release/git` asset. If we don't do this, the dependency churn would
   be committed into the `chore(release)` commit alongside the real version bump.
2. It cannot collide with the consumer's own dependency tree.

The plugins still resolve because, with no `extends` in the generated config,
semantic-release resolves plugin names relative to its own install location before falling
back to the working directory. The version bump and release commit are unaffected: both
`@semantic-release/npm` and `@semantic-release/git` operate on the working directory, which
is the consumer's checkout.

## Ordering

The Docker build runs *inside* semantic-release rather than as a separate workflow step,
and the plugin order in the generated config is load-bearing:

| Step | Plugin order | What happens |
|---|---|---|
| `prepare` | `npm` → `exec` → `git` | version written to `package.json`, **then** image built, then the release commit pushed |
| `publish` | `exec` → `github` | image tagged and pushed, then the GitHub release created |

Two properties follow from this.

**The image contains the correct version.** `@semantic-release/npm` writes the new version
into `package.json` during `prepare`, and only then does `exec` build the image. Building as
an ordinary workflow step — before semantic-release runs — would bake in the *previous*
version, which matters for any service that reads its own version at runtime.

**A failed build publishes nothing.** If `docker build` fails during `prepare`,
`@semantic-release/git` never runs, so no release commit is pushed. semantic-release creates
the git tag only after `prepare` completes, so there is no tag and no GitHub release either.
The only mutation is to `package.json` in the runner's throwaway checkout.

This is why the build does not use `docker/build-push-action`: that action cannot know the
version, because semantic-release has not computed it yet at step level.

## Changes take effect immediately

Consumers track `@main` by design. A bad commit here breaks every consumer's next run,
with no pinning to fall back on. Verify changes before pushing!
