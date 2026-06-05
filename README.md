> [!WARNING]
> **`busbar-actions` is under heavy active development — expect breaking changes.**
> These repositories are public, but **not ready for use yet** — please don't depend on them.
> A pilot is starting soon: **[star and watch the busbar-actions organization](https://github.com/busbar-actions)** for the launch of Discussions and the pilot announcement.

# busbar-actions/sf-anonymize-extract

Extract data from a source Salesforce org, anonymize PII, and load it into a target sandbox — with residual-PII validation before load, a pre-load snapshot, and a backup-id for manual rollback.

Backed by the **dedicated `sf-anonymize-extract` binary** (NOT `busbar-sf`). It is typesynth-free: it consumes the schema-free `sf-dataset-core` (dataset types, CSV storage, the PII `DataValidator`, and the `fake`-based anonymization engine), `busbar-auth` (per-org OIDC self-mint of the source/target sessions), and `busbar-sf-bulk` (Bulk API 2.0 query + ingest). It replaces the never-implemented `busbar-sf data pipeline`.

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
| `source-instance` | yes† | `` | Instance URL of the SOURCE org (a Salesforce org with Busbar installed and trust established). The source session **self-mints its own short-lived token via GitHub OIDC**, bound to this org — no token handoff. Requires `permissions: id-token: write`. Mapped to `SF_SOURCE_INSTANCE_URL`. |
| `target-instance` | no\* | `` | Instance URL of the TARGET org (Busbar installed + trust established). The target session self-mints in-process via OIDC. Required unless `dry-run=true`. |
| `eca-client-id` | no | `` | Override the External Client App consumer key for the OIDC exchange. PBO-pinned default. |
| `token-handler` | no | `` | Override the Apex token-exchange handler dev name. Defaults to `BBGitHubTokenExchangeHandler`. |
| `oidc-audience` | no | `` | Override the audience requested in the GitHub OIDC token. Defaults to each leg's instance URL. |
| `source-access-token` | no | `` | **Local-dev override only.** Raw SOURCE org token; leave empty in CI so the source leg self-mints via OIDC. When set, the source leg skips OIDC. |
| `source-instance-url` | no | `` | **Local-dev override only.** SOURCE org instance URL paired with `source-access-token`. In CI use `source-instance`. |
| `target-access-token` | no | `` | **Local-dev override only.** Raw TARGET org token; leave empty in CI so the target leg self-mints via OIDC. |
| `target-instance-url` | no | `` | **Local-dev override only.** TARGET org instance URL paired with `target-access-token`. In CI use `target-instance`. |
| `config-file` | yes | — | Pipeline config YAML (mapping + anonymization + load). |

> † `source-instance` is required in CI (the source leg self-mints from it via OIDC); you may instead supply `source-instance-url` for the local-dev token path. \* `target-instance` is required unless `dry-run=true`. The binary self-mints **both** org sessions in-process via GitHub OIDC (`busbar-auth` `oidc_session_for`, bound per leg to that org); the `*-access-token` inputs are a local-dev override that bypasses OIDC for that leg. See the Auth model below.
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
  id-token: write                # each org leg self-mints via OIDC

jobs:
  refresh:
    runs-on: ubuntu-latest
    environment: sandbox-refresh   # gate behind a protected environment
    steps:
      - uses: actions/checkout@v4

      - uses: busbar-actions/sf-anonymize-extract@v1
        with:
          source-instance: ${{ vars.SF_PROD_INSTANCE_URL }}
          target-instance: ${{ vars.SF_SANDBOX_INSTANCE_URL }}
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
  id-token: write                # the source leg self-mints via OIDC

jobs:
  dry-run:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: busbar-actions/sf-anonymize-extract@v1
        with:
          source-instance: ${{ vars.SF_PROD_INSTANCE_URL }}
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

The action installs the prebuilt `sf-anonymize-extract` binary from `busbar-actions/actions-dist` (via `busbar-actions/setup`), then runs it once. The binary reads its `INPUT_*` inputs plus the two-org env (`SF_SOURCE_INSTANCE_URL` for extract, `SF_TARGET_INSTANCE_URL` for load) and **self-mints each leg's token in-process via GitHub OIDC** (`SF_SOURCE_ACCESS_TOKEN` / `SF_TARGET_ACCESS_TOKEN` are a local-dev override only). It performs extract → anonymize → residual-PII gate → snapshot → load, writes the report JSON, emits `GITHUB_OUTPUT`/step-summary/annotations itself, and exits non-zero on a residual-PII block or load failure.

Auth uses per-org `busbar-auth` OIDC sessions (`oidc_session_for`, bound per leg to that org's instance URL). See the Auth model below.

## Auth model (OIDC self-mint)

The Busbar security principle is that a Salesforce access token must NEVER be handed to a script, written to `GITHUB_ENV`, passed as an action input/output, or persisted — each binary mints its OWN short-lived token in-process from the runner's GitHub OIDC id-token, holds it only in zeroizing memory, and revokes + zeroizes it (`session.dispose()`) at exit. The sibling actions `sf-schema` and `sf-metadata-retrieve` do this with `busbar-auth` `session_from_env` (one org per job).

This action drives **two** orgs in one job, so it self-mints **both** legs independently via `busbar-auth` `oidc_session_for(<instance_url>)` — the per-instance variant that binds the OIDC exchange (and audience) to a specific org so a leaked token can't be replayed elsewhere:

- **Each leg self-mints via OIDC.** The source session is minted from `source-instance` and the target session from `target-instance`; no Salesforce token is ever handed to the binary. The caller job MUST grant `permissions: id-token: write`.
- **Both orgs must have Busbar installed and a trust relationship established** for the calling repo/workflow. An exchange with no matching trust rule fails the run with a pending-approval message — approve it in the org and re-run.
- `dispose()` is called on **both** sessions on success *and* on error; for an OIDC-minted leg it **revokes** the token at the org and zeroizes it. The extracted access-token string is held in `zeroize::Zeroizing` so its bytes are wiped on drop.
- **Local-dev override:** set `SF_SOURCE_ACCESS_TOKEN` / `SF_TARGET_ACCESS_TOKEN` (via the `*-access-token` inputs) to skip OIDC for that leg and use a raw token directly. Those sessions are zeroize-only on dispose (there's nothing of ours to revoke). Use this only for local/manual runs.

In CI, supply only the instance URLs and grant `id-token: write`:

```yaml
permissions:
  contents: read
  id-token: write          # REQUIRED: lets each org leg self-mint via OIDC

jobs:
  refresh:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: busbar-actions/sf-anonymize-extract@v1
        with:
          source-instance: ${{ vars.SF_PROD_INSTANCE_URL }}
          target-instance: ${{ vars.SF_SANDBOX_INSTANCE_URL }}
          config-file: .busbar/pipelines/sandbox-refresh.yml
```

## Manual rollback

`rollback-on-failure` does **not** restore the target automatically in v1. The `snapshot` step writes a real, restorable backup of the target's current records to `<output-dir>/target-backup/` (or creates a Salesforce `OrgSnapshot` when the target is a scratch org and `SF_TARGET_SCRATCH_ORG_ID` is set). On a partial load failure the run fails and surfaces that `backup-id` so an operator can restore the target by hand.
