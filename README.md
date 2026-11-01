# ci

The one pipeline every app of Jenn's uses. Public on purpose: it holds no secrets, and a public
repo is the simplest way for all her repos, public or private, to share it.

**What it does:** on a pull request it runs the app's tests, builds the container, starts it and
checks it answers. On a merge to `main` it does that again, publishes the image tagged with the
exact commit, and commits that tag into the app's `deploy/kustomization.yaml`. Argo CD watches
that file and deploys the new version within a few minutes. Nothing is ever deployed as `latest`.

**Using it in an app repo** (`.github/workflows/pipeline.yml`):

```yaml
name: pipeline
on:
  pull_request:
  push:
    branches: [main]
    paths-ignore: ["deploy/**", "**.md"]
permissions:
  contents: write
  packages: write
jobs:
  app:
    uses: jennpcfo/ci/.github/workflows/app.yml@<commit the v1 tag points at> # v1
    with:
      image: ghcr.io/jennpcfo/<repo>
      # port: 8080, health_path: /healthz, smoke_paths: "/", test_command: "npm test"
```

The app needs a `Dockerfile`, an endpoint that answers 200 (default `/healthz`), and
`deploy/kustomization.yaml` with an `images:` entry carrying `newTag`. `pourprofitcfo-site` is the
working example. Changes here go through a pull request; move the `v1` tag only after testing.

**Pinned, not trusting tags.** Every action here, and this workflow in each app, is pinned to a full
commit SHA with the version in a comment. A tag can be moved by whoever controls it; a commit
cannot. Dependabot (`.github/dependabot.yml`, weekly, one week cooldown) proposes the updates.
After moving `v1`, update each app's `uses:` line to the new commit.

**Deploys only move forward.** Pushes to `main` run one at a time per repo, and the deploy step
pins a commit only if the one already pinned is its ancestor. A re-run of an old run does
nothing. To roll back, revert on `main` (or edit `newTag` by hand in a pull request).
