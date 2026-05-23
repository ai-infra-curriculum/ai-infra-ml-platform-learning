# Module 06: Model Management — Quiz

15 questions. 75% pass.

### 1. The registry is the source of truth for:
- [x] a) What's in production
- [ ] b) Training data
- [ ] c) Cost attribution
- [ ] d) Feature freshness

### 2. Stage transitions are typically audit-logged because:
- [x] a) "Who promoted v6 to prod?" must be answerable later
- [ ] b) It's required by Kubernetes
- [ ] c) MLflow won't work otherwise
- [ ] d) For UI display

### 3. Rolling deployment:
- [x] a) Progressive replacement of old with new
- [ ] b) Full duplicate then cutover
- [ ] c) Auto-revert canary
- [ ] d) Shadow mirroring

### 4. Blue-Green deployment:
- [x] a) Full duplicate; atomic traffic flip
- [ ] b) 5% → 25% → 50% → 100%
- [ ] c) Mirror without serving
- [ ] d) Progressive replacement

### 5. Canary:
- [x] a) Small initial percentage with auto-gate on metrics
- [ ] b) Bulk traffic to new version
- [ ] c) Two parallel models permanently
- [ ] d) Always 50/50

### 6. Shadow:
- [x] a) 0% real traffic; new model receives mirror only
- [ ] b) Cron-driven swap
- [ ] c) Always 100% to new
- [ ] d) Old + new alternate

### 7. When auto-gate fails on canary:
- [x] a) Argo rolls back automatically; registry untouched
- [ ] b) Manual rollback required
- [ ] c) New version becomes Production
- [ ] d) Both versions serve traffic

### 8. A model bypass-loading pattern (joblib.load from disk) violates:
- [x] a) Registry source-of-truth contract
- [ ] b) GDPR
- [ ] c) MIT license
- [ ] d) RFC 2616

### 9. Governance gates at promotion typically require:
- [x] a) Model card + bias review + decision log + audit entry
- [ ] b) GPU benchmarks
- [ ] c) Container image scan
- [ ] d) Helm chart

### 10. Aliases (`champion`, `challenger`) in the registry:
- [x] a) Pointers to versions; allow zero-downtime rotation
- [ ] b) Same as tags
- [ ] c) Override stages
- [ ] d) Required by Kubernetes

### 11. Atomic rollback requires:
- [x] a) Transition current → Archived AND prev → Production in one action
- [ ] b) Just demoting current
- [ ] c) Manual intervention
- [ ] d) Image rebuild

### 12. Audit log entries should record:
- [x] a) Timestamp, actor, from/to versions, reason, gate values
- [ ] b) Just timestamp
- [ ] c) Just actor
- [ ] d) Latency metrics only

### 13. Quarterly compliance check verifies:
- [x] a) Every Production model has current cards + bias reviews
- [ ] b) GPU utilization
- [ ] c) Network policies
- [ ] d) Helm values

### 14. Production model loading at serving startup should:
- [x] a) Pull from the registry by name+stage; not from disk paths
- [ ] b) Always use the most recent image
- [ ] c) Train fresh
- [ ] d) Cache forever

### 15. If a deployed model performs worse than offline metrics suggested:
- [x] a) Suspect feature skew or registry/serving mismatch
- [ ] b) The model is bad; retrain
- [ ] c) Add more replicas
- [ ] d) Rebuild image

---

Answers: all `a`
