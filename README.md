# spike-dp-analysts-v2

## Approach 2: Centralized Bundle — Pure dbt Code with Cross-Repo Dispatch

This repo contains **only dbt code and CI**. There is no Databricks Asset Bundle or CD pipeline here — all deployment is handled by [spike-dp-core-v2](https://github.com/vikas-tesco-mobile/spike-dp-core-v2).

### What this repo contains
- `dbt_project/` — Full dbt project (gold layer models sourcing from silver tables)
- `.github/workflows/ci.yml` — CI pipeline (dbt unit tests, pre-commit hooks, cross-repo dispatch)
- No `databricks.yml`, no `cd.yml`, no workflows

### How it works
1. Analysts push dbt changes to this repo
2. CI runs dbt unit tests and linting
3. On merge to `main`, CI triggers `spike-dp-core-v2` CD via GitHub `repository_dispatch`
4. Core repo re-deploys all bundles (including dbt workflows that reference this repo via `git_source`)

### Relationship
```
spike-dp-analysts-v2 (this repo)          spike-dp-core-v2
┌─────────────────────┐                   ┌──────────────────────┐
│ dbt_project/        │                   │ src/dp_core/         │
│ CI only             │──dispatch──────>  │ workflows/ (all)     │
│ No bundle           │                   │ configs/             │
│ No CD               │                   │ databricks.yml       │
└─────────────────────┘                   │ CI + CD              │
                                          │ dbt via git_source   │
                                          └──────────────────────┘
```

### CI Pipeline
- **pre-commit**: dbt-checkpoint (deps, parse, check-model), sqlfluff (lint, fix)
- **dbt unit tests**: `dbt test --select test_type:unit --target dev`
- **cross-repo dispatch**: On merge to main, triggers core-v2 CD pipeline

### Local Development
```bash
poetry install --with dev
cd dbt_project
poetry run dbt deps
poetry run dbt test --select test_type:unit --target dev
```
