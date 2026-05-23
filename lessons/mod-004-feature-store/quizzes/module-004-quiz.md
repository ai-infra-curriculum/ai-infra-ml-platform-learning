# Module 04: Feature Store Architecture — Quiz

- 15 questions • 25 min • 75% pass

### 1. The primary problem feature stores solve is:
- [x] a) Training-serving skew + feature reuse across teams
- [ ] b) Model deployment
- [ ] c) GPU scheduling
- [ ] d) Cluster autoscaling

### 2. The three layers of a feature store are:
- [x] a) Offline store + online store + registry
- [ ] b) Training + serving + monitoring
- [ ] c) Data + model + serving
- [ ] d) Schema + storage + compute

### 3. Point-in-time correctness ensures:
- [x] a) Features are retrieved as of the label timestamp, not the current time
- [ ] b) Features are always within the last minute
- [ ] c) Features include only timestamps
- [ ] d) Training and serving use different feature values

### 4. The online store typically uses:
- [x] a) Redis / DynamoDB / Bigtable (sub-10ms reads)
- [ ] b) Parquet on S3
- [ ] c) Postgres only
- [ ] d) The same store as offline

### 5. Materialization is the process of:
- [x] a) Copying latest values from offline → online store
- [ ] b) Training the model
- [ ] c) Creating new features
- [ ] d) Deleting old features

### 6. A common training-serving skew root cause is:
- [x] a) Feature computed via SQL at training, recomputed in Python at serving
- [ ] b) Different model architectures
- [ ] c) Different GPUs
- [ ] d) Different schedulers

### 7. The Feast Entity object represents:
- [x] a) A thing features describe (user, item, session)
- [ ] b) A model
- [ ] c) A dataset
- [ ] d) A serving endpoint

### 8. FeatureView TTL controls:
- [x] a) How stale a feature can be before considered missing at serving
- [ ] b) How long the feature definition persists
- [ ] c) Online store capacity
- [ ] d) Training duration

### 9. Tumbling vs sliding window:
- [x] a) Tumbling = non-overlapping; sliding = overlapping (more compute)
- [ ] b) Tumbling is always better
- [ ] c) They're identical
- [ ] d) Sliding is always real-time

### 10. Lambda architecture for streaming features:
- [x] a) Batch (accurate, historical) + streaming (low-latency, recent)
- [ ] b) Streaming-only
- [ ] c) Batch-only
- [ ] d) Required by GDPR

### 11. When is a feature store NOT worth introducing?
- [x] a) Single team, single model, static features
- [ ] b) Multiple teams sharing features
- [ ] c) Mix of streaming + batch features
- [ ] d) Regulated industry

### 12. Materialization frequency is typically:
- [x] a) Every 5-30 minutes
- [ ] b) Daily only
- [ ] c) Real-time always
- [ ] d) Weekly

### 13. The right approach to late-arriving events in streaming features is:
- [x] a) A configurable lateness window (e.g., 10 min) + batch reconciliation
- [ ] b) Drop them
- [ ] c) Wait forever
- [ ] d) Process synchronously

### 14. Feature drift is monitored using:
- [x] a) Per-feature PSI computed daily vs a baseline
- [ ] b) Manual review
- [ ] c) Log scanning
- [ ] d) Model accuracy only

### 15. When a streaming feature stalls (no new data):
- [x] a) Disable it in serving; fall back to a batch baseline
- [ ] b) Keep serving stale values
- [ ] c) Crash the model
- [ ] d) Block all predictions

---

Answer key: all `a`
