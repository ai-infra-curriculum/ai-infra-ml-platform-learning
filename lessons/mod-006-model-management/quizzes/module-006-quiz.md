# Module 06: Model Management — Quiz

15 questions. 75% pass.

### 1. The model registry is the source of truth for:
- [ ] a) Training data lineage
- [ ] b) Per-model cost attribution
- [ ] c) Feature freshness
- [x] d) Which model version is in each environment

### 2. Stage transitions are audit-logged primarily because:
- [ ] a) MLflow requires it
- [ ] b) It's needed for UI display
- [ ] c) Required by Kubernetes admission controllers
- [x] d) "Who promoted v6 to prod, when, and why?" must be answerable months later

### 3. Rolling deployment is best described as:
- [x] a) Progressive pod-by-pod replacement of old with new in a single Deployment
- [ ] b) Full duplicate of the fleet, then atomic cutover
- [ ] c) Canary with auto-revert
- [ ] d) Mirror traffic to a non-serving instance

### 4. Blue-Green deployment:
- [ ] a) Progressive replacement of pods within one Deployment
- [ ] b) Gradual percentage ramp (5%→25%→50%→100%)
- [ ] c) Mirroring traffic without serving responses
- [x] d) Full second fleet stood up; atomic traffic flip via Service selector

### 5. Canary deployment:
- [ ] a) Bulk shift to new version at midnight
- [ ] b) Two parallel models permanently serving 50/50
- [ ] c) Always splits exactly 50/50
- [x] d) Small initial traffic % with auto-gate on real-time metrics

### 6. Shadow deployment:
- [x] a) 0% real traffic; the new model receives mirror copies only
- [ ] b) Cron-driven nightly swap
- [ ] c) 100% to new; old version retained for rollback
- [ ] d) Old + new alternate per request

### 7. When the canary auto-gate fails:
- [ ] a) Manual rollback required from the on-call
- [ ] b) The new version automatically becomes Production
- [ ] c) Both versions serve traffic indefinitely
- [x] d) Argo Rollouts auto-reverts; the model registry is untouched

### 8. Loading a model with `joblib.load("/data/model.pkl")` from disk violates:
- [x] a) Registry-as-source-of-truth (the version is implicit + unauditable)
- [ ] b) GDPR data minimization
- [ ] c) The MIT license terms
- [ ] d) RFC 2616 (HTTP)

### 9. Governance gates at Production promotion typically require:
- [ ] a) GPU performance benchmarks
- [ ] b) Container image vulnerability scan only
- [ ] c) Helm chart validation
- [x] d) Model card + bias review + decision log + audit entry

### 10. Registry aliases (`champion`, `challenger`):
- [x] a) Mutable pointers to specific versions; enable zero-downtime rotation
- [ ] b) Synonyms for tags
- [ ] c) Override the version's stage
- [ ] d) Required by Kubernetes

### 11. Atomic rollback to a prior Production version requires:
- [ ] a) Just demoting the current version to Archived
- [ ] b) Manual coordination via Slack
- [ ] c) Rebuilding the serving image
- [x] d) Demoting current → Archived AND promoting prev → Production in one transactional action

### 12. A useful audit log entry for a stage transition records:
- [x] a) Timestamp, actor, from/to versions, reason, gate metric values
- [ ] b) Just the timestamp
- [ ] c) Just the actor's user id
- [ ] d) Latency + RPS at the moment of promotion

### 13. The quarterly compliance check verifies:
- [ ] a) GPU utilization stays above 80%
- [ ] b) NetworkPolicies are in place
- [ ] c) Helm values are valid
- [x] d) Every Production model has a current model card + bias review

### 14. Production model loading at serving startup should:
- [ ] a) Use whichever model file is in the image
- [ ] b) Train fresh on container start
- [ ] c) Cache the model forever; never re-fetch
- [x] d) Pull from the registry by (name, stage); not from arbitrary disk paths

### 15. A model that scores well offline but poorly online — first thing to suspect:
- [x] a) Feature skew (training-time vs serving-time computation diverges)
- [ ] b) The model architecture is wrong; retrain from scratch
- [ ] c) Scale up replicas
- [ ] d) Rebuild the image

---

## Answer key + rationale

1. **d** — Registry's job is to answer "what version is where right now". Lineage / cost / freshness are tracked elsewhere.
2. **d** — Production stage transitions are governance-critical events.
3. **a** — Rolling is the K8s default Deployment strategy.
4. **d** — Blue-green stands up the second fleet first; the cutover is just changing the Service selector.
5. **d** — Canary's distinguishing feature is the automated gate on real metrics.
6. **a** — Shadow's whole point is risk-free validation.
7. **d** — Argo handles the rollback in the serving layer; the registry's Production pointer never moves during a failed canary.
8. **a** — `joblib.load("/data/model.pkl")` means nobody can tell which version is in prod without reading the pod.
9. **d** — These are the standard governance artifacts; missing any of them is a promotion blocker.
10. **a** — Aliases are mutable pointers; rotating an alias is one API call with zero downtime.
11. **d** — Transactional both-sides means no window where both versions are Production (or neither is).
12. **a** — Five fields cover who/what/when/why/with-what-evidence.
13. **d** — The quarterly check is fundamentally a governance freshness check.
14. **d** — Tying serving to the registry makes "what's in prod" a single source of truth.
15. **a** — Skew is the #1 cause of "offline ≠ online" performance gaps; the second-most-common is mismatched feature versions.
