# Project 04 — Architecture

## Component map

```
┌──────────────────────────────────────────────────────────────┐
│ USER LAYER                                                   │
│  • Python SDK / CLI                                          │
│  • Training pipeline (auto-registers models)                 │
│  • Promotion approvers (humans)                              │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────┐
│ REGISTRY                                                      │
│  ┌──────────────────┐   ┌──────────────────┐                │
│  │ Metadata API     │   │ Lineage tracker  │                │
│  │ (CRUD + search)  │   │                  │                │
│  └────────┬─────────┘   └──────────────────┘                │
│           │                                                  │
│           ▼                                                  │
│  ┌──────────────────────────────────────────┐               │
│  │ PostgreSQL                                │               │
│  │  • models                                 │               │
│  │  • model_versions                         │               │
│  │  • lineage_edges                          │               │
│  │  • promotions                             │               │
│  │  • deployments                            │               │
│  └──────────────────────────────────────────┘               │
│                                                              │
│  Signatures + signatures attestations: Sigstore (Cosign).   │
│  Artifact storage: S3 / object store (only digest + URI     │
│    stored here).                                             │
└──────────────────────────────────────────────────────────────┘
```

## Key design decisions

### 1. Metadata-only registry; artifacts in object storage

The registry stores metadata + digests + signatures, not the
model bytes. Artifacts live in S3 / object storage, addressed
by content hash. This keeps the registry fast and the storage
cheap.

### 2. Immutable versions

A `ModelVersion` is immutable once registered. Promoting to
production never modifies the version; it adds a `Deployment`
that points to it.

### 3. Promotion is a state machine

States: `Registered → Staging → Production → Deprecated → Decommissioned`.
Each transition emits an audit event with the approver's
signed approval (where applicable).

### 4. Deployment is a separate resource

A `Deployment` ties a `ModelVersion` to a target environment +
rollout strategy. Multiple deployments per version are
allowed (e.g., same model deployed to multiple tenants).

### 5. Lineage as a DAG

`lineage_edges` is an edge list: model → (training data, feature
store snapshot, base model if fine-tuned, training run ID).
Reverse traversal answers "what models are affected by this
data?"

## Data model

```sql
CREATE TABLE models (
    id UUID PRIMARY KEY,
    name TEXT NOT NULL,
    namespace TEXT NOT NULL,
    owner TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    UNIQUE (namespace, name)
);

CREATE TABLE model_versions (
    id UUID PRIMARY KEY,
    model_id UUID REFERENCES models(id),
    version TEXT NOT NULL,
    artifact_uri TEXT NOT NULL,             -- s3://...
    artifact_digest TEXT NOT NULL,          -- sha256:...
    signature_uri TEXT NOT NULL,            -- Cosign signature blob
    metadata JSONB NOT NULL,                -- training metadata
    metrics JSONB NOT NULL,                 -- accuracy, fairness, robustness
    status TEXT NOT NULL,                   -- Registered|Staging|Production|Deprecated|Decommissioned
    registered_by TEXT NOT NULL,
    registered_at TIMESTAMPTZ NOT NULL,
    UNIQUE (model_id, version)
);

CREATE TABLE lineage_edges (
    model_version_id UUID REFERENCES model_versions(id),
    source_kind TEXT NOT NULL,              -- 'training_data', 'feature_snapshot', 'base_model', 'training_run'
    source_identifier TEXT NOT NULL,
    PRIMARY KEY (model_version_id, source_kind, source_identifier)
);

CREATE TABLE promotions (
    id UUID PRIMARY KEY,
    model_version_id UUID REFERENCES model_versions(id),
    from_status TEXT NOT NULL,
    to_status TEXT NOT NULL,
    approver TEXT NOT NULL,
    approved_at TIMESTAMPTZ NOT NULL,
    approval_signature TEXT NOT NULL,       -- signed by approver's identity
    audit_chain_entry_id UUID NOT NULL
);

CREATE TABLE deployments (
    id UUID PRIMARY KEY,
    model_version_id UUID REFERENCES model_versions(id),
    target_environment TEXT NOT NULL,        -- 'staging-us-west-2', 'prod-eu-west-1'
    rollout_strategy TEXT NOT NULL,         -- 'rolling', 'blue-green', 'canary', 'shadow'
    traffic_share NUMERIC,                   -- for canary: 0.05 = 5%
    deployed_at TIMESTAMPTZ NOT NULL,
    deployed_by TEXT NOT NULL,
    status TEXT NOT NULL                     -- Active|Rolled-back|Decommissioned
);
```

## Promotion gates

The default gates for `Staging → Production`:

```yaml
gates:
  - name: signature
    condition: signature_valid AND signing_identity_matches(expected_workflow)
  - name: accuracy
    condition: metrics.accuracy >= 0.85
  - name: fairness
    condition: metrics.fairness.disparate_impact <= 1.25
  - name: robustness
    condition: metrics.adversarial.robust_accuracy_at_eps_8_255 >= 0.30
  - name: human_approval
    type: approval
    approvers: [team-lead, security-eng]
```

Gates configurable per model (high-stakes models add more
gates; low-stakes models can skip some with documented
justification).

## Rollout strategies

| Strategy | Behavior |
|---|---|
| **Rolling** | Replace old version with new, one pod at a time. Default. |
| **Blue-green** | Bring up new fleet in parallel, swap traffic at LB. |
| **Canary** | Send a configurable % of traffic to new version, monitor metrics, ramp or rollback. |
| **Shadow** | Send 100% of traffic to old + shadow to new; compare outputs offline. |

Rollback: `smartrecs deployments rollback <deployment-id>` —
points the target back to the prior `Production` version.

## Lineage queries

- **Forward** (this model is built from what?): standard.
- **Reverse** (what models are affected by this dataset?):
  needed for "we discovered a data quality issue in
  dataset-v3.2; which models trained on it?"

Implementation: SQL recursive CTE or graph traversal. For the
capstone scale, a recursive CTE is fine.

## Multi-tenancy

- Models live in tenant namespaces.
- The platform exposes only the tenant's own models via the
  list API.
- Cross-tenant model sharing is explicit: a `share` action
  records the share + audit event; the sharer must be the
  model's owner.

## Cross-references

- Module 06 lecture notes for the conceptual treatment.
- [project-03-workflow-orchestration](../project-03-workflow-orchestration/) for the gate framework integration.
- Security track Module 10 for the supply-chain controls (Cosign + attestations).
