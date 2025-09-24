---
title: "Ship Databricks Workloads with DABs — Part 1: The Essentials"
layout: post
excerpt: "Kickstart your CI/CD-ready Databricks projects with Databricks Asset Bundles (DABs). This first installment shows how to scaffold with the CLI, init, validate and deploy."
last_modified_at: 2025-09-23T10:11:03-05:00
tags:
  - Databricks 
  - DABs
  - Databricks Asset Bundles
  - Spark 
  - Python
  - CLI
  - Big Data
  - YML
---

# The DAB () Story: What They Are and Why They Matter
Before we touch the template, here’s the quick origin story. **Databricks Asset Bundles (DABs)** are Databricks’ native way to manage your workspace assets _as code_. They showed up in the GA on  **April 23, 2024** to answer a simple question:  
**“How do we keep Databricks jobs, pipelines, and data objects consistent across dev → prod without duct tape?”**

## Life before DAB (a familiar mess)

-   **Notebook shuffle**: exporting/importing notebooks by hand.
-   **CLI-only deploys**: better than click-ops, but still lots of brittle scripts.
-   **Fragile, hand-crafted JSON**: custom job/pipeline JSON, copied per environment.
-   **Drift & mystery**: dev, stage, and prod quietly diverge.

## What DAB brings to the table

> **One YAML, one lifecycle, many environments.**

-   **YAML-first definition**: A single `databricks.yml` declares your resources—Jobs, Lakeflow/Delta Live Pipelines, Unity Catalog objects (schemas/volumes), permissions, variables, and more.
-   **Organized Structure**: A The structure is designed to scale with your project needs and keep code organized like src, resources, test, docs etc.
-   **Targets for environments**: `dev`, `stage`, `prod` live in the same file with their own hosts, paths, identities, and overrides.
-   **Single flow**: `validate → deploy → run` via the Databricks CLI—identical locally and in CI/CD.
-   **Adopt, don’t rewrite**: Generate bundle YAML from existing jobs/pipelines and bring them under IaC.
-   **Less drift by design**: Everything lands in Git; workspaces are reconciled by the bundle.

# When should I use Databricks Asset Bundles?

Use **DABs** when you want your Databricks workload to behave like software you can ship—versioned, reviewable, and reproducible across environments.

### Great fit (use DABs when…)
-   **You have multiple environments** (dev/stage/prod) and need consistent, repeatable promotion.
-   **CI/CD matters**: you want PR-based changes, automated validation, deploys, and runs.
-   **You manage many assets**—Jobs, Lakeflow/Delta Live Pipelines, Unity Catalog schemas/volumes, permissions—and want them declared in one place.
-   **You need drift control**: changes made in the UI should be reconciled back to what’s in Git.
-   **You’re standardizing projects**: new teams should be able to clone a repo, set a target, deploy, and go.
-   **You want environment-specific settings** (hosts, root paths, identities, parameters) without forking YAML.
-   **You’re adopting existing click-configured assets** and want to bring them under IaC (generate YAML, then manage via bundles).
-   **Auditability/governance is required**: who changed what, when, and where needs to be reviewable.

## How DABs fit with Terraform (rule of thumb)

-   **Terraform**: platform plumbing (accounts, workspaces, networks, pools, secret scopes, instance profiles).
-   **DABs**: workload plumbing (jobs, pipelines, UC schemas/volumes, permissions, code deployment, parameters).
- They’re complementary: platform once, workloads per project.

# How do I get started with bundles?

You’ve got two clean entry points—pick the one that matches your comfort level:

### Option A — Databricks CLI (Git-first)

-   **Best for:** folks comfortable with Git/VS Code and planning CI/CD.
-   **What you do (high level):** install the CLI → authenticate → `bundle init` from a template → commit `databricks.yml` → `bundle validate` → `bundle deploy` → `bundle run`.
-   **Why choose this:** PR reviews, repeatable deployments, and an easy path to promotion across dev/stage/prod.
    

### Option B — Databricks Workspace UI (click-first)

-   **Best for:** beginners or quick prototypes without local setup.
-   **What you do (high level):** create a new bundle from the UI → choose a template → (optionally) connect a repo → fill in environment basics → deploy.
-   **Why choose this:** fast onboarding and visual scaffolding before you standardize on Git-based flows.
    
**Rule of thumb:** production-bound projects, use the **Databricks CLI only** (Git-first, CI/CD-friendly).

# Quickstart: Scaffold a Bundle via CLI

## `bundle init`

Use `bundle init` to scaffold a new Databricks Asset Bundle from a template—fastest way to start a real project.

#### What it does
Initializes a repo with a ready-to-run `databricks.yml` plus starter code, so you can immediately `validate → deploy → run`.

#### Syntax

`databricks bundle init [TEMPLATE_PATH] [--output-dir <DIR>]` 

-   `TEMPLATE_PATH` (optional) chooses which template to use. It can be:
    -   `default-python` — Default Python template for notebooks and Lakeflow-style pipelines
    -   `default-sql` — SQL-first template for `.sql` files running on Databricks SQL
    -   `dbt-sql` — dbt + Databricks SQL template (see: databricks.com/blog/delivering-cost-effective-data-real-time-dbt-and-databricks)
    -   `mlops-stacks` — Databricks MLOps Stacks (github.com/databricks/mlops-stacks)
    -   `experimental-jobs-as-code` — Experimental “jobs as code” template
    -   A **local path** to a template directory
    -   A **Git URL** to a template repo (e.g., `https://github.com/my/repository`)
-   `--output-dir` writes the scaffold into a specific folder (defaults to current directory).
#### Examples
```bash
# Pick from built-in templates via an interactive prompt
databricks bundle init

# Create a Python jobs/notebooks project
databricks bundle init default-python

# Create a dbt + SQL Warehouse project
databricks bundle init dbt-sql
```
## Folder structure created (what you’ll see)
Here’s what you typically get **immediately after** `databricks bundle init` with the default Python template—just the scaffold the template generates:
```bash
<project-name>/
├── databricks.yml                   # Bundle config
├── pyproject.toml  
├── resources
├── fixtures
├── tests
├── scratch
└── src
```

# Validate and deploy the bundle

## Validate the bundle

`validate` is your preflight check. It reads `databricks.yml` (plus any files it references), resolves variables and the selected target, and fails fast if something’s off.
```bash
# Validate against the default target (if one is marked default: true)
databricks bundle validate

# Or validate a specific environment
databricks bundle validate -t dev
databricks bundle validate -t prod

# Validate with on-the-fly variable overrides
databricks bundle validate -t dev --var catalog=hive_metastore --var bronze_schema=bronze
```

## Deploy the bundle

`deploy` takes the desired state in your repo and **applies it to a workspace target**—creating or updating the resources declared in `databricks.yml`.
```yml
# This is a Databricks asset bundle definition for FirstDAB.
# See https://docs.databricks.com/dev-tools/bundles/index.html for documentation.
bundle:
  name: FirstDAB
  uuid: 31549c16-4b5a-458d-9053-3afe6ce5c996

include:
  - resources/*.yml
  - resources/*/*.yml

targets:
  dev:
    # The default target uses 'mode: development' to create a development copy.
    # - Deployed resources get prefixed with '[dev my_user_name]'
    # - Any job schedules and triggers are paused by default.
    # See also https://docs.databricks.com/dev-tools/bundles/deployment-modes.html.
    mode: development
    default: true
    workspace:
      host: https://<dev workspace URL>

    presets:
      # Set dynamic_version: true on all artifacts of type "whl".
      # This makes "bundle deploy" add a timestamp to wheel's version before uploading,
      # new wheel takes over the previous installation even if actual wheel version is unchanged.
      # See https://docs.databricks.com/aws/en/dev-tools/bundles/settings
      artifacts_dynamic_version: true

  prod:
    mode: production
    workspace:
      host: https://<prod workspace URL>
      # We explicitly deploy to /Workspace/Users/<user_id> to make sure we only have a single copy.
      root_path: /Workspace/Users/xyz@abc.com/.bundle/${bundle.name}/${bundle.target}
    permissions:
      - user_name: <user_name>
        level: CAN_MANAGE
```


```bash
# Deploy to the default target (if one is marked default: true)
databricks bundle deploy

# Deploy to a specific environment
databricks bundle deploy -t dev
databricks bundle deploy -t prod

# Deploy with on-the-fly variable overrides
databricks bundle deploy -t dev \
  --var catalog=hive_metastore \
  --var bronze_schema=bronze

```
# Deploy a workflow job & modularize the bundle config
Keep your bundle clean by **splitting resources into focused files** (jobs, UC objects, perms), then deploy with the same CLI flow.

### 1) Modularize: split resources under `resources/`

In your top-level `databricks.yml`, keep **targets**, **variables**, and an `include:` that pulls in everything from `resources/`:

```yml
bundle:
  name: LearnDAB

include:
  - resources/*.yml
  - resources/*/*.yml

variables:
  catalog:         { default: hive_metastore }
  bronze_schema:   { default: bronze }
  silver_schema:   { default: silver }
  gold_schema:     { default: gold }

targets:
  dev:
    default: true
    workspace:
      host: https://<dev-workspace>
  prod:
    workspace:
      host: https://<prod-workspace>
      root_path: /Workspace/Users/<you>/.bundle/${bundle.name}/${bundle.target}
```

### 2) Create a workflow job (modular file)

`resources/jobs/medallion_etl.yml`

```yml
resources:
  jobs:
    medallion_etl:
      name: ${bundle.name}-etl
      job_clusters:
        - job_cluster_key: etl_cluster
          new_cluster:
            spark_version: "13.3.x-scala2.12"
            node_type_id: "i3.xlarge"          # adjust per cloud/workspace
            num_workers: 2
      tasks:
        - task_key: bronze_ingest
          job_cluster_key: etl_cluster
          notebook_task:
            notebook_path: ./notebooks/bronze/ingest.py
            base_parameters:
              catalog: ${var.catalog}
              bronze_schema: ${var.bronze_schema}

        - task_key: silver_transform

        - task_key: gold_aggregate

      schedule:
        quartz_cron_expression: "0 0 3 * * ?"   # 3 AM daily
        timezone_id: "<Time Zone>"
      permissions:
        - user_name: <you@company.com>
          level: CAN_MANAGE

```
### Why modularize?

-   **Separation of concerns:** jobs, notebooks, UC objects, and (optionally) permissions live in their own files.
-   **Simpler diffs & reviews:** small, targeted YAML changes per PR.
-   **Reusable patterns:** copy `resources/jobs/*.yml` files to start new workloads quickly.
-   **Same lifecycle:** validation, deployment, and runs don’t change—just your file organization.
    
> Keep **environment differences** in `targets` and **`variables`**—avoid per-environment copies of job YAML.

# How Databricks tracks bundle deployment state

-   **Workspace-scoped state (source of truth).**  
    Each deploy records the **resource IDs** (jobs, pipelines, etc.) that your bundle created, and stores this state **in the workspace file system** (not in Git). On the next deploy, the CLI updates those same IDs in place—so renaming a job in YAML won’t create a duplicate; it updates the existing one.
    
-   **State location & identity.**  
    By default, bundle state lives under a workspace path derived from your bundle’s root; you can customize this with `settings.state_path` (defaults to something like `${workspace.root}/state`). The effective identity is **workspace + bundle name + target (and root path)**—keep these stable per environment. 
    
-   **Local cache in `.databricks/` (speeds up deploys).**  
    The CLI also keeps a **project-local cache** (e.g., `.databricks/`) with files such as a **`sync_snapshot` JSON** and a **`deployment.json`**. These track timestamps or checksums of uploaded artifacts so unchanged files are skipped on subsequent deploys—useful for fast, incremental uploads. (This cache **is not** the authoritative state; the workspace state is.)
    
-   **Idempotent & drift-correcting.**  
    Because it matches by IDs, re-running `bundle deploy` is idempotent and will **reconcile** manual UI edits back to the YAML-defined shape for resources managed by the bundle. 
    
-   **Tear-down when needed.**  
    When you’re done, `databricks bundle destroy` deletes previously deployed jobs, pipelines, and artifacts for that bundle identity—use with care.
