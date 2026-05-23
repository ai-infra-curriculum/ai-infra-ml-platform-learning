# Module 05: Workflow Orchestration — Quiz

15 questions. 75% pass.

### 1. The default orchestrator for most ML platforms is:
- [x] a) Airflow
- [ ] b) Cron
- [ ] c) Lambda only
- [ ] d) Kubeflow

### 2. XCom should hold:
- [x] a) Paths/IDs/small metadata; not data blobs
- [ ] b) Full pandas DataFrames
- [ ] c) Model weights
- [ ] d) Container logs

### 3. DAG-per-model fits when:
- [x] a) Few models with substantially different shapes
- [ ] b) Hundreds of similar models
- [ ] c) Irregular event-driven triggers
- [ ] d) Always

### 4. Parametric DAGs fit when:
- [x] a) Many models with similar shape but different config
- [ ] b) Few specialized models
- [ ] c) Single-tenant clusters
- [ ] d) Real-time inference

### 5. Event-driven orchestration fits when:
- [x] a) Data arrival is irregular; cron would waste resources
- [ ] b) Always
- [ ] c) Only for daily jobs
- [ ] d) When budget is unlimited

### 6. Continuous training fits when:
- [x] a) Drift-sensitive models where stale model cost > retrain cost
- [ ] b) Slow-drift environments
- [ ] c) Single model
- [ ] d) Manual approvals required

### 7. Airflow pool capacity vs parallelism:
- [x] a) Pool = per-pool task cap; parallelism = cluster-wide cap
- [ ] b) Same thing
- [ ] c) Pool > parallelism always
- [ ] d) Neither matters

### 8. Backfill safety requires (pick all):
- [x] a) Audit log
- [x] b) Concurrency cap
- [x] c) Dry-run mode
- [x] d) Idempotency
- [ ] e) Mandatory full-table re-backfill

### 9. SLA miss callback fires when:
- [x] a) A DAG run exceeds its declared SLA
- [ ] b) Any task fails
- [ ] c) Scheduler is unhealthy
- [ ] d) Pool is exhausted

### 10. When pool slots are exhausted, the right first action is:
- [x] a) Identify the slow tasks holding slots; investigate
- [ ] b) Resize pool immediately
- [ ] c) Disable retries
- [ ] d) Kill long-running tasks

### 11. Storing model weights in XCom causes:
- [x] a) Slow scheduler, large metadata DB, hard-to-debug serialization issues
- [ ] b) Faster training
- [ ] c) Better reproducibility
- [ ] d) Improved logging

### 12. KubernetesPodOperator is the right choice when:
- [x] a) The task should run as a separate pod with its own image + resources
- [ ] b) All tasks
- [ ] c) Only quick (< 10s) tasks
- [ ] d) Streaming features

### 13. Five-thousand DAGs without organizing principle:
- [x] a) Indicates a missing parametric pattern or platform-template approach
- [ ] b) Is fine
- [ ] c) Required for scale
- [ ] d) Solved by faster scheduler

### 14. The right alert for "every task failure" is:
- [x] a) No alert; retries are normal — alert on 3 consecutive failures
- [ ] b) Slack DM every failure
- [ ] c) Page on-call every failure
- [ ] d) Email the whole team

### 15. Choosing an orchestrator by "what we read about most recently" is:
- [x] a) An anti-pattern; choose by fit + maturity + hiring pool
- [ ] b) Best practice
- [ ] c) Common but acceptable
- [ ] d) Required for innovation

---

Answers: 1.a 2.a 3.a 4.a 5.a 6.a 7.a 8.a+b+c+d 9.a 10.a 11.a 12.a 13.a 14.a 15.a
