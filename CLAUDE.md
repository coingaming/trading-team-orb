# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

`odds88/tradeart-orb` — the CircleCI orb that holds every TradeArt build/deploy pipeline step. It contains no application code; it is authored in *unpacked* form under [src/](src/) and packed into a single `orb.yml` at publish time (`orb.yml` is gitignored — never commit it).

Consumer repositories import this orb and compose its jobs into their own workflows, so **any change here affects every TradeArt service pipeline**. Treat parameter renames, removals, and default changes as breaking.

## Common commands

```bash
circleci orb pack src > orb.yml                       # pack unpacked source into a single orb
circleci orb validate orb.yml                         # validate the packed orb locally
circleci orb publish orb.yml odds88/tradeart-orb@dev:alpha  # publish a dev version (expires in 90 days)
yamllint src                                          # config in .yamllint (relaxed, max line length 200)
bats src/tests                                        # run the BATS tests for src/scripts/*.sh
```

## Publishing

A production release is cut by pushing a tag matching `^v[0-9]+\.[0-9]+\.[0-9]+$` (see `release-filters` in [.circleci/test-deploy.yml](.circleci/test-deploy.yml)). Current line is `v4.x`. Semantic versioning — bump major for any breaking parameter change.

The pipeline is a two-stage setup workflow:
1. [.circleci/config.yml](.circleci/config.yml) (`setup: true`) — lint, review (`RC009,RC010` excluded), pack, shellcheck, then publish a `dev:` version and hand off via `orb-tools/continue`.
2. [.circleci/test-deploy.yml](.circleci/test-deploy.yml) — runs integration tests against the just-published dev orb; on a release tag it also runs `PROD publish`.

Integration tests for new commands go into `test-deploy.yml`'s `jobs:` section and must be wired into the `test-deploy` workflow with `filters: *filters`, and added to the `requires:` list of `PROD publish` — otherwise they don't gate the release.

## Source layout and composition model

`src/@orb.yml` is the orb root: description, display URLs, and imported orbs (`circleci/slack@5.1.1` as `slack`, `guitarrapc/git-shallow-clone` as `git-shallow-clone`). At pack time each directory becomes a top-level key, with the filename as the element name (`src/jobs/build.yml` → job `build`).

- [src/executors/](src/executors/) — `default` (.NET SDK image `trading-dev-core-sdk`, tag parameterized, working dir `/mnt/ramdisk`) and `docker-builder` (`trading-dev-docker-build:4.0`, used for anything touching Docker/Helm/AWS).
- [src/commands/](src/commands/) — the reusable steps that do real work.
- [src/jobs/](src/jobs/) — the public surface consumers use. Jobs are thin: pick an executor, declare parameters, call commands, then notify Slack.
- [src/scripts/](src/scripts/) — shell bodies inlined via `<<include(scripts/x.sh)>>`; the only part covered by BATS tests.

### Conventions that hold across nearly every job

- **Job = executor + parameters + commands + Slack notify.** Almost every job ends with `inject_slack_templates` followed by `slack/notify` on `event: fail` (and `event: pass` for deploys) to `odds88-circleci-notifications`. Keep that tail when adding jobs.
- **`executor_class` parameter** (default `medium`, `small` for lightweight jobs) is threaded into the executor's `resource_class`.
- **`builder_image_tag`** (default `'3.1'`) selects the `default` executor image tag; `test-with-db` uses `'5.0-db'`, `intent_check` pins `"10.0"`.
- **`inject_version_data`** must run before anything that references `$FULL_VERSION`. It writes `VERSION_PREFIX=<base_version>.<pipeline_id>`, a branch-derived `VERSION_SUFFIX` (`master` → `test`), and `FULL_VERSION=$VERSION_PREFIX-$VERSION_SUFFIX` into `$BASH_ENV`. Because these are `$BASH_ENV` exports, `$FULL_VERSION` is passed to later commands as the literal string `$FULL_VERSION`, resolved at shell time — not as a config-time value.
- **Build → Docker handoff** goes through the workspace: `build`/`lambda_build` `persist_to_workspace` the whole tree; `docker`/`lambda_docker` `attach_workspace` at `./`. Docker images are built from a `CircleDockerfile` (not `Dockerfile`) in the consumer repo, multi-arch (`linux/amd64,linux/arm64`) via buildx, pushed to ECR `178636341549.dkr.ecr.eu-central-1`.

### Branch → environment mapping

This is hardcoded in shell across several commands and is the single most important thing to keep consistent when editing them:

| Branch | Env name | Global namespace | Tenant namespace | Cluster vars |
|---|---|---|---|---|
| `master` | TEST | `trading-test` | `tenant-<id>-dev` | `DEV_CLUSTER_NAME` / `TEST_CLUSTER_VALUES_NAME` |
| `staging` | STAGE | `trading-stage` | `tenant-<id>-stage` | `STAGING_*` |
| `release` | PROD | `trading-prod` | `tenant-<id>-prod` | `PROD_*` |

Note the asymmetry: [deploy_using_helm.yml](src/commands/deploy_using_helm.yml) defaults values to `TEST_CLUSTER_VALUES_NAME` while [deploy_using_helm_tenanted.yml](src/commands/deploy_using_helm_tenanted.yml) defaults to `DEV_CLUSTER_VALUES_NAME`. Any new env-aware command must reproduce all three branches.

### External repositories the orb clones at runtime

Several commands clone sibling private repos over SSH using a deploy key echoed into `~/.ssh/id_rsa_1`:

- `coingaming/trading-circleci-notifications` (`GITHUB_DEPLOY_PRIVATE_KEY`) — Slack message templates, the `users_map.json` CircleCI-user → Slack-tag map, and the shared `helm/` chart. Cloned by `checkout_common_project`, `inject_slack_templates`, `lint_helm*`.
- `coingaming/tradeart-terraform` (`GITHUB_TERRAFORM_PRIVATE_KEY`) — per-tenant Helm values at `terraform/clients/pool1/tenant-<id>/`. Cloned by `checkout_terraform_project`.
- `coingaming/tradeart-tenants` / `tradeart-tenants-prod` (`DEV_CLUSTER_REPO_KEY` / `PROD_CLUSTER_REPO_KEY`) — ArgoCD GitOps repos; `update_argo_image_*` `sed`s the image tag and pushes a commit.
- `coingaming/tradeart-intent-modules` (`GITHUB_INTENT_PRIVATE_KEY`) — Intent Architect modules for `intent_check`.

### Two deployment paths

- **Helm push** (`helm`, `helm-tenanted`): `connect_to_vpn` opens an SSH tunnel through `vpn.trading.io` to the K8s API endpoint, then `helm upgrade --install --atomic`. Validated ahead of time by `validate-helm` / `validate-helm-tenanted` (helm lint against the same values file).
- **ArgoCD GitOps** (`argo_deploy_dev`, `argo_deploy_prod`): no cluster access; commits a new image tag to the tenants repo. `deployment_type: global` edits `argocd/services/overlays/<env>/<svc>/version.yaml`; `tenanted` edits `argocd/base/.../deployment.yaml`.

### Testing jobs

`test` and `test-with-db` both shallow-clone (`--filter=blob:none`) then `git fetch --no-filter --refetch` — Sonar needs full history. `docker_layer_caching` is opt-in (default `false`) because it bills a flat ~200 credits and these jobs run no docker build. `dotnet_test` runs the SonarCloud scanner (org `coingaming`); `dotnet_test_no_sonar` / `test_no_sonar` are the fork for repos without a Sonar project. Both convert `.trx` to JUnit via `trx2junit` before `store_test_results`.
