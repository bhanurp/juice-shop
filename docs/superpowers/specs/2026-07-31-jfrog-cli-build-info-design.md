# JFrog CLI + Build-Info Integration for GitHub Actions Workflows

## Goal

Enhance the repository's GitHub Actions workflows to produce [buildinfo.org](https://www.buildinfo.org/)-compliant build-info for every package ecosystem the project builds, using JFrog CLI's `setup-jfrog-cli` action and the ["ghost/frog" package-alias interception](https://docs.jfrog.com/artifactory/docs/jfrog-cli-package-alias) mechanism, rather than rewriting every command to its `jf <tool>` equivalent.

## Repo composition (confirmed)

This is a pure npm/Node.js monorepo — no other language ecosystems exist (no Go/Python/Java/Ruby manifests found). "Build info for all languages" therefore means build-info for:
- **npm (root)** — `package.json` name `juice-shop`
- **npm (frontend)** — `frontend/package.json` name `frontend`
- **Docker** — the single image built from `./Dockerfile`

## Scope

Workflows touched: `ci.yml`, `release.yml`, `frontend-bundle-analysis.yml`. All other workflows (CodeQL, ZAP scan, lint-fixer, stale, rebase, lock, update-challenges-*, update-news-*, image_actions) are untouched — they don't build or publish artifacts.

Within `ci.yml`/`release.yml`, only artifact-producing jobs get full build-info + publish: `smoke-test`/`package` (npm) and `docker`. Pure test-matrix jobs (`lint`, `frontend-test`, `server-test`, `api-test`, `custom-config-test`, `e2e-test`) are deliberately left unchanged — running full build-info collection across a 3×3 OS/Node matrix would create noisy, duplicate builds with no traceability benefit, since none of those jobs publish anything.

## New secrets (placeholders — a maintainer must provision these against a real JFrog Platform instance)

- `JF_URL` — JFrog Platform base URL
- `JF_ACCESS_TOKEN` — access token with deploy rights to the repos below

## New Artifactory repos (placeholder names, to be created by a maintainer)

- `juice-shop-npm-virtual` — virtual repo (npmjs remote + local) used to **resolve** npm dependencies
- `juice-shop-npm-local` — local repo used to **deploy** the packaged tarball
- `juice-shop-docker-local` — local Docker repo used to deploy a copy of the image for build-info/scanning

## Mechanism: ghost/frog package-alias interception

Per the JFrog CLI package-alias docs, the flow is:

```shell
jf npm-config --repo-resolve=juice-shop-npm-virtual
jf package-alias install --packages=npm
echo "$HOME/.jfrog/package-alias/bin" >> $GITHUB_PATH
echo "JFROG_CLI_GHOST_FROG=true" >> $GITHUB_ENV
```

After this one-time setup, **existing** `npm install` invocations in the job are transparently intercepted and routed through `jf npm install` — no need to rewrite them to `jf npm install` explicitly. Build identity is attached via `--build-name=X --build-number=Y` flags appended to the existing install command (or equivalently `JFROG_CLI_BUILD_NAME`/`JFROG_CLI_BUILD_NUMBER` env vars). The same mechanism applies to Docker via `jf package-alias install --packages=docker`, intercepting plain `docker build`/`docker push` invocations.

Each build is finalized with:
```shell
jf rt build-add-git <name> <number>
jf rt build-publish <name> <number>
```

Two independent builds are used per workflow run (avoids merging build-info across separate jobs/runners, which JFrog CLI does not support without shared storage):
- **`juice-shop-npm`** — two modules (root `juice-shop`, frontend `frontend`), both captured within the same job so they merge into one build.
- **`juice-shop-docker`** — the Docker image module.

Build number: `github.run_number` for `ci.yml` and the bundle-analysis workflow; the git tag for `release.yml` (so release build-info is keyed by version, not CI run count).

## Per-workflow changes

### `ci.yml`

**`smoke-test` job** (already runs `npm install --production` then `npm run package:ci`, producing `dist/juice-shop-*.tgz`):
1. Add `setup-jfrog-cli` step (after checkout).
2. Add `jf npm-config --repo-resolve=juice-shop-npm-virtual` + `jf package-alias install --packages=npm` + PATH/env wiring.
3. Append `--build-name=juice-shop-npm --build-number=${{ github.run_number }}` to the existing `npm install --production` line. No separate frontend install step is needed: root `package.json`'s `postinstall` hook already runs `cd frontend && npm install` as a nested child process, which inherits the job's shimmed `PATH` and `JFROG_CLI_GHOST_FROG`/build-name/build-number env vars, so it gets intercepted the same way and registers as the second (`frontend`) module of the same build. **Assumption to confirm on first live run** (no JFrog instance available to verify here): that a nested/child-process `npm install` invoked from a postinstall script is captured by the alias shim exactly like a top-level one. If it isn't, the fallback is to skip scripts on the root install and add an explicit `cd frontend && npm install --production --build-name=... --build-number=...` step, restoring the build steps postinstall would otherwise have done.
4. After `npm run package:ci` produces the tarball, `jf rt upload` it into `juice-shop-npm-local`.
5. `jf rt build-add-git juice-shop-npm ${{ github.run_number }}` then `jf rt build-publish juice-shop-npm ${{ github.run_number }}`.

**`docker` job** (already builds+pushes multi-arch to Docker Hub via `docker/build-push-action`):
1. Leave the existing multi-arch Docker Hub push **completely unchanged** — buildx's interaction with the ghost/frog docker interception is undocumented for multi-registry/multi-platform pushes, so we don't risk the proven Docker Hub release path on an unverified mechanism.
2. Add `setup-jfrog-cli` + `jf package-alias install --packages=docker` + PATH/env wiring.
3. Add a **separate**, plain, single-arch (`linux/amd64`) `docker build -t <JF_URL-host>/juice-shop-docker-local/juice-shop:${{ env.DOCKER_TAG }} .` followed by `docker push <same-ref> --build-name=juice-shop-docker --build-number=${{ github.run_number }}` — these are plain CLI invocations matching the documented intercepted-command list exactly.
4. `jf rt build-add-git` + `jf rt build-publish juice-shop-docker ${{ github.run_number }}`.
5. **Known tradeoff, called out explicitly:** this rebuilds the image a second time (single-arch) purely for Artifactory build-info/traceability, adding CI time. This is the safe/documented choice over an unverified combined-push trick.

### `release.yml`

Same treatment, scoped to avoid redundant work:
- **`package` job**: apply the npm build-info steps only on the `ubuntu-latest` + default Node-version matrix leg (mirrors the existing pattern of restricting side-effect steps to one leg). Build number = the release tag.
- **`docker` job**: same treatment as `ci.yml`'s docker job. Build number = the release tag.

### `frontend-bundle-analysis.yml`

Add `setup-jfrog-cli` + `jf npm-config` + `jf package-alias install --packages=npm` before the existing backend/frontend `npm install --ignore-scripts` steps, with build name `juice-shop-frontend-bundle-analysis` / build number `${{ github.run_number }}`. This workflow doesn't publish anything (it only commits a screenshot), so no `jf rt upload` step is added — build-info here captures the dependency graph only, as a lightweight second example of build-info in a non-publishing workflow.

## What stays unchanged

Docker Hub push, Heroku deploy, Slack notifications, CodeQL, ZAP scan, and every other workflow/job not listed above.

## Validation plan

No live JFrog Platform instance is available in this environment to test against. Validation consists of:
1. YAML syntax validation of the edited workflow files.
2. Manual cross-check of every `jf`/`jfrog` command and flag against the `jfrog-cli-package-alias` documentation fetched during design.
3. This will be called out explicitly as **unverified against a real JFrog instance** in the final summary — a maintainer with real `JF_URL`/`JF_ACCESS_TOKEN` secrets and the placeholder repos created will need to confirm a live run succeeds.
