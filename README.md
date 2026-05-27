# busbar-actions/sf-anonymize-extract

Extract data from a source Salesforce org, anonymize PII, and load it into a target sandbox — with residual-PII validation before load, a pre-load snapshot, and rollback on failure.

Built on the `sf-datasets` pipeline: `Pipeline` executor, `TransformChain` with `fake`-crate anonymization, `BackupManager` / `RollbackManager` snapshots, and the PII `validator`.

## What it does

Wraps `busbar-sf data pipeline`, driven by a YAML config that defines the extraction mapping, anonymization rules, and load settings. The flow:

1. **Extract** from the source org (Bulk API 2.0, dependency-ordered).
2. **Anonymize** per the config's transform rules (`fake`-generated names/emails/etc., value mappings, masking).
3. **Validate residual PII** on the anonymized output. If PII at or above `residual-pii-fail-on` remains, the load is **aborted** — anonymized data that still contains PII never reaches the target.
4. **Snapshot** the target objects (so the run is reversible).
5. **Load** into the target org.
6. **Roll back** to the snapshot if the load partially fails (when enabled).

## Safety model

- **Two orgs, clearly separated**: `source-*` (read) and `target-*` (write) credentials.
- **Residual-PII gate before load** is on by default — the load only proceeds if the anonymized data is clean.
- **Dry-run** extracts + anonymizes + validates without touching the target.
- **Snapshot + rollback** on by default so a failed load doesn't leave the target in a partial state.
- **Data is not uploaded as an artifact by default** — even anonymized data can be sensitive. Only the run report is.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `source-access-token` | yes | — | SOURCE org token (extract from). |
| `source-instance-url` | yes | — | SOURCE org instance URL. |
| `target-access-token` | no* | `` | TARGET org token (load into). *Required unless `dry-run=true`. |
| `target-instance-url` | no* | `` | TARGET org instance URL. *Required unless `dry-run=true`. |
| `config-file` | yes | — | Pipeline config YAML (mapping + anonymization + load). |
| `dry-run` | no | `false` | Extract + anonymize + validate, but don't load. |
| `snapshot` | no | `true` | Snapshot target objects before load. |
| `rollback-on-failure` | no | `true` | Roll target back to snapshot on partial load failure. |
| `validate-residual-pii` | no | `true` | Validate anonymized data before load; abort on breach. |
| `residual-pii-fail-on` | no | `high` | Residual-PII severity that aborts the load. |
| `output-dir` | no | `.busbar/pipeline-runs` | Where data + report are written. |
| `upload-artifact` | no | `true` | Upload the run report. |
| `artifact-name` | no | `pipeline-run` | Artifact name. |
| `upload-data` | no | `false` | Also upload extracted/anonymized data (off by default). |
| `version` | no | `latest` | `busbar-sf` release tag. |
| `binary-repo` | no | `busbar-actions/actions-dist` | Where to fetch the binary. |

## Outputs

| Output | Description |
|---|---|
| `status` | `success`, `dry-run`, `residual-pii-blocked`, `failed`, or `rolled-back`. |
| `operation-id` | Pipeline operation ID. |
| `backup-id` | Snapshot ID for manual rollback (empty if no snapshot). |
| `records-extracted` | Records extracted from source. |
| `records-loaded` | Records loaded into target. |
| `records-failed` | Records that failed to load. |
| `residual-pii-count` | PII findings remaining at or above the threshold. |
| `report-path` | Path to the run report JSON. |

## Example: refresh a sandbox on demand

```yaml
name: Refresh Sandbox (Anonymized)

on:
  workflow_dispatch:
    inputs:
      dry-run:
        description: 'Extract + anonymize only, do not load'
        type: boolean
        default: false

permissions:
  contents: read

jobs:
  refresh:
    runs-on: ubuntu-latest
    environment: sandbox-refresh   # gate behind a protected environment
    steps:
      - uses: actions/checkout@v4

      - uses: busbar-actions/sf-anonymize-extract@v1
        with:
          source-access-token: ${{ secrets.SF_PROD_ACCESS_TOKEN }}
          source-instance-url: ${{ secrets.SF_PROD_INSTANCE_URL }}
          target-access-token: ${{ secrets.SF_SANDBOX_ACCESS_TOKEN }}
          target-instance-url: ${{ secrets.SF_SANDBOX_INSTANCE_URL }}
          config-file: .busbar/pipelines/sandbox-refresh.yml
          dry-run: ${{ inputs.dry-run }}
```

## Example: dry-run on PR to validate the pipeline config

```yaml
name: Validate Pipeline Config

on:
  pull_request:
    paths:
      - '.busbar/pipelines/**'

permissions:
  contents: read

jobs:
  dry-run:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: busbar-actions/sf-anonymize-extract@v1
        with:
          source-access-token: ${{ secrets.SF_PROD_ACCESS_TOKEN }}
          source-instance-url: ${{ secrets.SF_PROD_INSTANCE_URL }}
          config-file: .busbar/pipelines/sandbox-refresh.yml
          dry-run: true
```

## Pipeline config

The `config-file` is a `sf-datasets` `PipelineConfig` YAML — extraction mapping, `AnonymizationPattern` rules per field, value mappings, and load settings. See `crates/sf-datasets/src/pipeline/config.rs` for the schema.

## Dependencies (current status)

This action is fully scaffolded but **not yet runnable end-to-end**. Pieces that need to land:

1. **`busbar-sf data pipeline` subcommand** — the `sf-datasets` pipeline pieces (`Pipeline`, `TransformChain`, `BackupManager`, `RollbackManager`, `validator`) exist as library; today's CLI only exposes the individual `data extract` / `data load` steps. The full orchestration needs a single command:

   ```
   busbar-sf data pipeline \
     --config <pipeline.yml> \
     --output <dir> \
     --report <report.json> \
     [--dry-run] \
     [--snapshot] \
     [--rollback-on-failure] \
     [--validate-residual-pii --residual-pii-fail-on <severity>] \
     [--json]
   ```

   Dual-org auth via env: `SF_SOURCE_ACCESS_TOKEN` / `SF_SOURCE_INSTANCE_URL` (extract) and `SF_TARGET_ACCESS_TOKEN` / `SF_TARGET_INSTANCE_URL` (load). **All SF API interactions must go through `busbar_sf_api`**; metadata/types via `busbar_sf_types`.

   The command must enforce the residual-PII gate *before* any load, write the existing `PipelineResult` shape to `--report` (plus `status` and `residual_pii_count` fields the action reads), and exit non-zero on `failed` / `residual-pii-blocked` / `rolled-back`.

2. **Binary publication to `busbar-actions/actions-dist`** — same dependency as the other actions.

Once both land, tag this action `v1` and consumers can pin it.
