# spi-images

This project contains the base images for Docker-based builds of the SwiftPackageIndex Build System.

The builds are processed as follows (in abbreviated form): 

```
docker run $IMAGE_NAME swift build
```

The `basic` image provides Linux builds, while the other variants add to that to provide build environments also for Android and for Wasm.

## 21-DOT-DEV: GitHub-hosted images

This fork additionally publishes images to GitHub Container Registry, because the upstream GitLab registry no longer allows anonymous pulls:

- `ghcr.io/21-dot-dev/spi-images:wasm-6.1.3` (and `:wasm-6.1-latest`, plus dated `:wasm-6.1.3-YYYYMMDD-<sha>` rollback tags)
- `ghcr.io/21-dot-dev/spi-images:android-6.1.3` (and `:android-6.1-latest`, plus dated `:android-6.1.3-YYYYMMDD-<sha>` rollback tags)

```bash
docker pull ghcr.io/21-dot-dev/spi-images:wasm-6.1.3
```

Only the wasm 6.1.3 and android 6.1.3 images are built here — `.github/workflows/publish-images.yml` mirrors the corresponding entries in `.gitlab-ci.yml` (same base image, SDK URL, and checksum). All other upstream files, including `.gitlab-ci.yml` and the `Dockerfile.*` files, are intentionally left unmodified so this fork stays easy to sync.

### One-time setup

- Create the `ghcr` branch carrying this fork's own commits (workflows, this README section, Dependabot config) and set it as the repo's **default branch** — scheduled workflows only run on the default branch.
- Enable GitHub Actions on this fork and re-enable scheduled workflows (forks start with schedules disabled).
- Enable **Issues** on the fork — they're off by default on forks and the drift report needs them (`gh api -X PATCH repos/21-DOT-DEV/spi-images -F has_issues=true`).
- If org rulesets or branch protection cover `main`/`ghcr`, they must allow the Actions token to push (`ghcr`) and force-push (both) — otherwise every sync run fails red.
- **After the first publish, set the `spi-images` package visibility to Public** in its package settings. New organization packages default to private, and anonymous pulls — which `swift-secp256k1`'s CI will need once it switches to this registry — fail until this is done.
- Note that GitHub disables scheduled workflows after 60 days with no repository activity.

### Branch model and upstream sync

- `main` is a **pure upstream mirror** — `upstream-sync.yml` force-updates it to upstream's head weekly. Never commit to it.
- `ghcr` is the default branch and carries the fork's own commits, **rebased onto upstream's head** each week and force-pushed with `--force-with-lease`.

`.github/workflows/upstream-sync.yml` runs weekly: it fetches upstream's GitHub mirror, diffs what a rebase would bring into `ghcr` (measured from the merge base, not `main` — so GitHub's "Sync fork" button can't skip review), reports build-input drift, and gates on `Dockerfile.wasm`/`Dockerfile.android-6.1` changes — those build steps execute inside `publish-images.yml`, so they require human review and the run fails red instead of pushing. `main` is updated last, on success. Review the drift issue, then resolve by hand:

```bash
git remote add upstream https://github.com/SwiftPackageIndex/spi-images  # once
git fetch upstream main && git checkout ghcr
git rebase upstream/main          # review the incoming changes here
git push --force-with-lease
```

Ordinary rebase conflicts fail red the same way. To decline an upstream change, commit a revert on `ghcr`; it carries forward through every rebase. When upstream's delta touches `.gitlab-ci.yml` or the Dockerfiles, the job opens or comments on a `sync-watch` issue with the changed lines so the matrix in `publish-images.yml` can be mirrored — close the issue once done.

Because `ghcr` is force-pushed weekly, open PRs into it (including Dependabot's action bumps) can show stale diffs afterward — rebase the PR branch before merging; Dependabot rebases its own PRs on its next run.

To build another image on this fork: add a matrix entry in `publish-images.yml` (dockerfile, `SWIFT_IMAGE`, `SDK_URL`, `CHECKSUM`, tags, smoke-test patterns) copied from the matching `.gitlab-ci.yml` case, add the Dockerfile to the workflow's path filters, and add the file to the watched-files list in `upstream-sync.yml`.
