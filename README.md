# busbar-actions/sf-anonymize-extract

Extract data from a source Salesforce org, anonymize PII, and load it into a target sandbox — with residual-PII validation before load, a pre-load snapshot, and a backup-id for manual rollback.

Backed by the **dedicated `sf-anonymize-extract` binary** (NOT `busbar-sf`). It is typesynth-free: it consumes the schema-free `sf-dataset-core` (dataset types, CSV storage, the PII `DataValidator`, and the `fake`-based anonymization engine), `busbar-auth` (explicit source/target sessions), and `busbar-sf-bulk` (Bulk API 2.0 query + ingest). It replaces the never-implemented `busbar-sf data pipeline`.

## What it does

Driven by a YAML config that defines the extraction mapping, anonymization rules, and load settings. The flow:

1. **Extract** from the source org (Bulk API 2.0).
2. **Anonymize** per the config's transform rules (`fake`-generated names/emails/etc.).
3. **Validate residual PII** on the anonymized output. If PII at or above `residual-pii-fail-on` remains, the load is **aborted** — anonymized data that still contains PII never reaches the target.
4. **Snapshot** the target objects to a local backup (so the run is reversible), or a Salesforce `OrgSnapshot` if the target is a scratch org (`SF_TARGET_SCRATCH_ORG_ID`).
5. **Load** into the target org (Bulk insert).
6. **Manual rollback**: on a partial load failure with `rollback-on-failure=true`, the run surfaces `backup-id` and fails — an operator restores the target by hand from the backup. **Automated in-place restore is out of scope for v1.**

## Safety model

- **Two orgs, clearly separated**: `source-*` (read) and `target-*` (write) credentials.
- **Residual-PII gate before load** is on by default — the load only proceeds if the anonymized data is clean.
- **Dry-run** extracts + anonymizes + validates without touching the target.
- **Snapshot** on by default so a failed load can be undone — but rollback is **manual** (restore from `backup-id`); v1 does not perform an automated in-place restore.
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
| `rollback-on-failure` | no | `true` | On partial load failure, surface the `backup-id` for MANUAL rollback (v1: no automated restore). |
| `validate-residual-pii` | no | `true` | Validate anonymized data before load; abort on breach. |
| `residual-pii-fail-on` | no | `high` | Residual-PII severity that aborts the load. |
| `output-dir` | no | `.busbar/pipeline-runs` | Where data + report are written. |
| `upload-artifact` | no | `true` | Upload the run report. |
| `artifact-name` | no | `pipeline-run` | Artifact name. |
| `upload-data` | no | `false` | Also upload extracted/anonymized data (off by default). |
| `version` | no | `latest` | `sf-anonymize-extract` release tag. |
| `binary-repo` | no | `busbar-actions/actions-dist` | Where to fetch the binary. |

## Outputs

| Output | Description |
|---|---|
| `status` | `success`, `dry-run`, `residual-pii-blocked`, or `failed`. |
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

The `config-file` is a focused YAML consumed by the `sf-anonymize-extract` binary (it does **not** reuse the entangled `sf-datasets` `PipelineConfig`). Each object lists the fields to extract and the per-field anonymization rules; fields without a rule are loaded verbatim:

```yaml
objects:
  - api_name: Contact
    # `fields` builds `SELECT Id, <fields> FROM Contact`. Id is always included.
    fields: [FirstName, LastName, Email, Phone]
    anonymization:
      FirstName: { pattern: first_name }
      LastName:  { pattern: last_name }
      Email:     { pattern: email }
      Phone:     { pattern: phone }
    # limit: 5000          # optional row cap for extraction
  - api_name: Account
    fields: [Name, Phone]
    anonymization:
      Name:  { pattern: company_name }
      Phone: { pattern: phone }

# Optional load options.
load:
  operation: insert        # insert | update | upsert
  # external_id_field: External_Id__c   # required for upsert
```

`pattern` values are the `AnonymizationPattern` variants from `sf-dataset-core` (`first_name`, `last_name`, `full_name`, `email`, `username`, `phone`, `street`, `city`, `state`, `country`, `zip_code`, `full_address`, `company_name`, `job_title`, `industry`, `url`, `domain_name`, `ip_address`, `sentence`, `paragraph`, `word`, `uuid`, `fixed`, `null`, …). See `crates/sf-dataset-core/src/anonymize.rs`.

> Note: raw `soql` is not supported — the safe Bulk `QueryBuilder` builds the SELECT from `fields`. Use `fields: [...]`.

## How it runs

The action installs the prebuilt `sf-anonymize-extract` binary from `busbar-actions/actions-dist` (via `busbar-actions/setup`), then runs it once. The binary reads its `INPUT_*` inputs plus the two-org env (`SF_SOURCE_ACCESS_TOKEN`/`SF_SOURCE_INSTANCE_URL` for extract, `SF_TARGET_ACCESS_TOKEN`/`SF_TARGET_INSTANCE_URL` for load), performs extract → anonymize → residual-PII gate → snapshot → load, writes the report JSON, emits `GITHUB_OUTPUT`/step-summary/annotations itself, and exits non-zero on a residual-PII block or load failure.

Auth uses explicit per-org `busbar-auth` sessions (`CredentialContext::new` + `SalesforceSession::new`), not `session_from_env` (which only reads a single `SF_ACCESS_TOKEN`).

### Manual rollback

`rollback-on-failure` does **not** restore the target automatically in v1. The `snapshot` step writes a real, restorable backup of the target's current records to `<output-dir>/target-backup/` (or creates a Salesforce `OrgSnapshot` when the target is a scratch org and `SF_TARGET_SCRATCH_ORG_ID` is set). On a partial load failure the run fails and surfaces that `backup-id` so an operator can restore the target by hand.
