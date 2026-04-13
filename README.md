# spike-dp-analysts-v2

## Approach 2: Centralized Bundle - Pure dbt Code With External Workspace Upload

This repo contains only dbt code and CI. There is no Databricks Asset Bundle or standalone CD pipeline here. Deployment is executed by uploading this repo's `dbt_project/` to a dedicated Databricks workspace folder, then running `databricks bundle deploy` from `spike-dp-core-v2`.

## What This Repo Contains

- `dbt_project/` -- Full dbt project for gold-layer models sourcing from silver tables
- `.github/workflows/ci.yml` -- CI pipeline for linting, dbt unit tests, core checkout, dbt staging, and deploy
- No `databricks.yml`
- No standalone workflow definitions
- No standalone CD pipeline

## How It Works

1. Analysts push dbt changes to this repo.
2. CI runs dbt unit tests and linting.
3. On merge to `main`, CI checks out `spike-dp-core-v2`.
4. CI uploads `dbt_project/` to a dedicated Databricks workspace folder outside DAB-managed bundle paths.
5. CI runs `databricks bundle deploy` from `spike-dp-core-v2`.
6. Core dbt workflows run from that external workspace dbt folder.

## Why this was introduced

This change was introduced because bundle deployment from `spike-dp-core-v2` is authoritative over the bundle-managed workspace path. When `dbt_project/` was staged inside the core bundle payload, CI/CD deploys could overwrite or remove the workspace copy if staging failed. Uploading the dbt project to a workspace folder outside DAB control avoids that overwrite behaviour.

## Relationship

```text
spike-dp-analysts-v2 (this repo)          spike-dp-core-v2
dbt_project/                              src/dp_core/
CI only                    ---------->    workflows/ (all)
No bundle                                  configs/
No standalone CD                           external workspace dbt_project
                                           databricks.yml
                                           bundle deploy
```

## CI Pipeline

- `pre-commit`: dbt-checkpoint, sqlfluff
- `dbt unit tests`: `dbt test --select test_type:unit --target dev`
- `cross-repo deploy`: on merge to `main`, uploads `dbt_project/` to Databricks workspace, then deploys `core-v2`

## Current Limitations

- Analysts repo is not independently deployable. Analysts can develop and unit-test models locally, but cannot validate deployed runtime behaviour from this repo because deployment is controlled by `spike-dp-core-v2`.
- Breaking changes are hard for analysts to detect. If bundle structure, workflow configuration, sync rules, or runtime paths change in core, analysts may be affected without visibility or control here.
- Local deployment from core is incomplete by default. The dbt project lives in a separate repo, so `databricks bundle deploy` from core requires staging `dbt_project/` first.
- Core bundle deploy and dbt project upload are separate lifecycle steps. This avoids DAB overwriting the dbt project, but makes deployment less atomic.
- Successful deployment does not guarantee a working dbt runtime. The bundle can deploy cleanly while dbt jobs fail at runtime if the staged dbt project is missing, empty, or incorrectly copied.
- Ownership becomes ambiguous. It is unclear whether the deployable source of truth for dbt lives in this repo or in the staged copy inside core.
- Neither repo is self-sufficient. Analysts cannot test deployment behaviour end-to-end, and core cannot test dbt workflows without pulling content from another repo.
- CI staging adds its own fragility. Replacing `git_source` with workspace-synced files removes the Databricks Git credential dependency, but introduces a new risk: the staged copy can drift from the source repo if the CI pipeline is not carefully maintained.

## Assessment

`v2` achieves centralized deployment at the cost of clarity, local autonomy, and runtime simplicity. The model works, but it is harder to reason about and more fragile than either a clean split-bundle approach (`v1`) or an explicit versioned-artifact approach (`v3`).

## Local Development

```bash
poetry install --with dev
cd dbt_project
poetry run dbt deps
poetry run dbt test --select test_type:unit --target dev
```
