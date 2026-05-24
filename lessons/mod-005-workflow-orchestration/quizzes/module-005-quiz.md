# Module 05: Workflow Orchestration — Quiz

15 questions. 75% pass.

### 1. The most common default orchestrator for ML platforms (largest community + hiring pool) is:
- [x] a) Apache Airflow
- [ ] b) cron + bash
- [ ] c) AWS Lambda functions only
- [ ] d) Kubeflow Pipelines

### 2. Airflow XCom is intended to hold:
- [ ] a) Full pandas DataFrames
- [ ] b) Model weights for downstream tasks
- [x] c) Small metadata — paths, IDs, scalar values; not data blobs
- [ ] d) Container logs

### 3. DAG-per-model is the right pattern when:
- [x] a) Few (~< 20) models with substantially different processing shapes
- [ ] b) Hundreds of similar models with the same shape
- [ ] c) Triggered by irregular S3 events
- [ ] d) Always, regardless of model count

### 4. Parametric DAGs fit best when:
- [ ] a) A handful of bespoke pipelines
- [ ] b) Single-tenant clusters
- [ ] c) Real-time inference traffic
- [x] d) Many (50+) models share a pipeline shape but differ in config

### 5. Event-driven orchestration shines when:
- [x] a) Data arrival is irregular and a fixed cron would either waste resources or miss data
- [ ] b) The job runs the same time every day
- [ ] c) The team only has daily batch needs
- [ ] d) Budget is unlimited so wake-ups are free

### 6. Continuous training is the right pattern when:
- [ ] a) The model rarely drifts
- [ ] b) Manual approval is required for every retrain
- [ ] c) The team has a single static model
- [x] d) The model is drift-sensitive AND cost-of-stale > cost-of-retrain

### 7. Airflow pool vs parallelism:
- [x] a) `pool.slots` = per-pool concurrent task cap; `parallelism` = cluster-wide concurrent task cap
- [ ] b) They mean the same thing
- [ ] c) Pool must always be greater than parallelism
- [ ] d) Neither matters at scale

### 8. Backfill safety requires (select all that apply):
- [x] a) Audit log per backfill operation
- [x] b) Concurrency cap (don't replay all 365 days in parallel)
- [x] c) Dry-run mode showing the execution plan
- [x] d) Idempotency (re-running produces same output)
- [ ] e) Mandatory full-table re-backfill on any new run

### 9. The Airflow `sla_miss_callback` fires when:
- [x] a) A DAG run's elapsed time exceeds the declared `sla` interval
- [ ] b) Any individual task in the DAG fails
- [ ] c) The scheduler heartbeat is lost
- [ ] d) A pool is exhausted

### 10. When pool slots are exhausted, the right first action is:
- [ ] a) Immediately resize the pool larger
- [ ] b) Disable task retries cluster-wide
- [ ] c) Kill all long-running tasks
- [x] d) Identify which tasks are holding slots and investigate *why* before scaling

### 11. Storing model weights in XCom causes:
- [ ] a) Faster downstream task startup
- [ ] b) Better experiment reproducibility
- [ ] c) Cleaner DAG logs
- [x] d) Scheduler slowdown, metadata DB bloat, and brittle serialization issues

### 12. `KubernetesPodOperator` is the right choice when:
- [x] a) The task should run as a separate pod with its own image, resources, and lifecycle
- [ ] b) Every task in every DAG (default to it)
- [ ] c) Only sub-10-second tasks
- [ ] d) Streaming features that never complete

### 13. A platform with 5,000 DAGs and no organizing principle indicates:
- [x] a) A missing parametric pattern or DAG-template library
- [ ] b) That the platform is operating well at scale
- [ ] c) That you need a faster scheduler
- [ ] d) Nothing — DAG count is incidental

### 14. The right alerting policy for "any task failure" is:
- [ ] a) Slack DM the team on every failure
- [ ] b) Page on-call on every failure (zero tolerance)
- [ ] c) Email the whole company
- [x] d) No alert — retries are normal; alert only on N consecutive failures or SLA miss

### 15. Choosing an orchestrator based on "what we read about most recently" is:
- [x] a) An anti-pattern; choose by fit + maturity + hiring pool + ops surface
- [ ] b) Standard industry practice
- [ ] c) Acceptable for greenfield teams
- [ ] d) Required to stay innovative

---

## Answer key + rationale

1. **a** — Airflow's community + hiring pool dominate; default choice unless you have a strong reason otherwise.
2. **c** — XCom serializes through the metadata DB; large payloads slow everything down. Pass S3 paths instead.
3. **a** — DAG-per-model scales linearly in code; only sane below ~20 models with diverse shapes.
4. **d** — Parametric works when shapes are similar; one template generates N pipelines.
5. **a** — Cron + irregular data = either wasted runs or missed data. Events solve both.
6. **d** — Continuous training has real cost; only worth it when staleness is more expensive.
7. **a** — Pool is per-pool; parallelism is cluster-wide. Both matter for capacity planning.
8. **a+b+c+d** — All four are required for a safe backfill story. "e" is wrong (don't over-backfill).
9. **a** — SLA miss is specifically about DAG run duration, not task failures.
10. **d** — Resizing without investigation hides the actual problem (slow task) until it surfaces again.
11. **d** — XCom is for paths/IDs/metadata; binary weights belong in a registry or S3.
12. **a** — KubernetesPodOperator lets each task have its own image + isolation; ideal for heterogeneous DAGs.
13. **a** — 5K DAGs without templates means everyone is copy-pasting. The fix is a template library.
14. **d** — Per-failure alerts cause alert fatigue. Retries usually solve transient errors.
15. **a** — Tools have long-tailed consequences. Choose for the next 3 years, not the trending HN post.
