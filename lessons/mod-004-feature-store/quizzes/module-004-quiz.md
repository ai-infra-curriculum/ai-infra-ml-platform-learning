# Module 04: Feature Store Architecture — Quiz

- 15 questions • 25 min • 75% pass

### 1. The primary problem feature stores solve is:
- [ ] a) Model deployment automation
- [ ] b) GPU scheduling at scale
- [ ] c) Cluster autoscaling under bursty load
- [x] d) Training-serving skew + cross-team feature reuse

### 2. The three architectural layers of a feature store are:
- [ ] a) Training + serving + monitoring
- [x] b) Offline store + online store + registry
- [ ] c) Data + model + serving runtimes
- [ ] d) Schema + storage + compute engines

### 3. Point-in-time correctness ensures:
- [x] a) Features are retrieved as of the label timestamp, not "now"
- [ ] b) All features are always within the last minute
- [ ] c) Only timestamp columns are joined
- [ ] d) Training and serving use different feature values

### 4. The online store typically uses:
- [ ] a) Parquet on S3 (high throughput, batch reads)
- [ ] b) Postgres for OLAP queries
- [ ] c) The same store as offline; consolidation simplifies ops
- [x] d) Redis / DynamoDB / Bigtable for sub-10ms key-value reads

### 5. Materialization is:
- [ ] a) The training-set creation step
- [ ] b) Promoting models from staging to production
- [ ] c) Deleting expired features beyond TTL
- [x] d) Copying latest values from the offline → online store

### 6. The most common training-serving skew root cause is:
- [x] a) Feature computed via SQL at training time, then recomputed in Python at serving time
- [ ] b) Different model architectures between training and serving
- [ ] c) Different GPUs in training vs serving
- [ ] d) Different schedulers

### 7. In Feast, an Entity represents:
- [ ] a) A model artifact
- [ ] b) A dataset version
- [x] c) The "thing" features describe (user, item, session) — the join key
- [ ] d) A serving endpoint

### 8. FeatureView TTL controls:
- [x] a) How stale a feature can be at serving time before it's treated as missing
- [ ] b) How long the feature *definition* persists in the registry
- [ ] c) Online store capacity
- [ ] d) Maximum training duration

### 9. Tumbling vs sliding windows:
- [ ] a) Tumbling is always better
- [ ] b) They are functionally identical
- [ ] c) Sliding windows are required for real-time use cases
- [x] d) Tumbling = non-overlapping (cheap); sliding = overlapping (more compute, more responsive)

### 10. Lambda architecture for streaming features means:
- [ ] a) Streaming-only with infinite Kafka retention
- [ ] b) Batch-only with hourly recomputation
- [ ] c) Required by GDPR Article 5
- [x] d) Batch (accurate, historical) + streaming (low-latency, recent) with reconciliation

### 11. When is a feature store NOT worth introducing?
- [ ] a) Multiple teams sharing features across products
- [ ] b) Mix of streaming + batch features
- [ ] c) Regulated industries with high cost-of-skew
- [x] d) Single team, single model, mostly static features

### 12. Materialization is typically scheduled:
- [x] a) Every 5-30 minutes
- [ ] b) Once per day after midnight
- [ ] c) Continuously (real-time) for every feature
- [ ] d) Weekly

### 13. The pragmatic approach to late-arriving streaming events is:
- [x] a) A configurable lateness window (e.g., 10 min) + batch reconciliation for the tail
- [ ] b) Drop them; they will skew aggregates
- [ ] c) Block the window until all events arrive
- [ ] d) Reprocess synchronously on each late arrival

### 14. Feature drift is operationally monitored using:
- [x] a) Per-feature PSI computed daily against a baseline window
- [ ] b) Manual review of feature distributions in notebooks
- [ ] c) Grep over inference logs
- [ ] d) Aggregate model accuracy only

### 15. When a streaming feature stalls (no new data ingesting):
- [x] a) Mark it stale in serving; fall back to a batch baseline value
- [ ] b) Keep serving the last seen value indefinitely
- [ ] c) Crash the model server
- [ ] d) Block all predictions until the stream recovers

---

## Answer key + rationale

1. **d** — Skew + reuse are the two value props. Deployment / scheduling / autoscaling are adjacent concerns.
2. **b** — Offline (training) + online (serving) + registry (metadata + lineage).
3. **a** — The label was generated at time T; the model can only "see" features known at time T.
4. **d** — Online stores are sub-10ms KV stores; latency dominates the design.
5. **d** — Materialization is the export step from offline → online.
6. **a** — Different code paths producing different feature values is the textbook skew root cause.
7. **c** — Entity is the join key abstraction; it's what features attach to.
8. **a** — Beyond TTL, the value at serving is treated as missing (which the consumer can handle gracefully).
9. **d** — Sliding overlaps and is more expensive; tumbling is the default unless you need overlap.
10. **d** — Lambda = batch + streaming + reconciliation. Kappa replaces both with streaming-only.
11. **d** — At single-model, single-team, static-feature scale, the operational overhead exceeds the benefit.
12. **a** — Sub-30-min materialization keeps online + offline in sync for most workloads.
13. **a** — Bounded lateness + batch fix-up is the practical middle ground.
14. **a** — PSI is the industry-standard drift metric; daily cadence balances signal vs noise.
15. **a** — Fallback to batch is safer than serving silently-stale data.
