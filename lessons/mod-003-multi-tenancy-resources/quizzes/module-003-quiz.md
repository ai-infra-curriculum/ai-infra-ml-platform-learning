# Module 03: Multi-Tenancy & Resource Management — Quiz

- 15 questions • 25 min • passing 75%

### 1. Default tenant isolation model for most ML platforms:
- [x] a) Namespace-per-team
- [ ] b) Shared namespace
- [ ] c) Cluster-per-team
- [ ] d) Virtual cluster

### 2. ResourceQuota enforces at:
- [x] a) Admission time (rejects pods exceeding the quota)
- [ ] b) Runtime only
- [ ] c) Scheduling time only
- [ ] d) Build time

### 3. LimitRange sets:
- [x] a) Default + max per-container resource values
- [ ] b) Cluster-wide quotas
- [ ] c) Network policies
- [ ] d) RBAC rules

### 4. Default-deny NetworkPolicy is best paired with:
- [x] a) Explicit allow policies for known traffic patterns
- [ ] b) Allowing all traffic
- [ ] c) Only egress restrictions
- [ ] d) Disabling DNS

### 5. Gang scheduling is needed for:
- [x] a) Distributed training jobs that need N pods running simultaneously
- [ ] b) Single-pod serving
- [ ] c) Cron jobs
- [ ] d) Backups

### 6. Volcano + Yunikorn add to the default scheduler:
- [x] a) Fair share, queue hierarchies, gang scheduling, preemption
- [ ] b) Node autoscaling
- [ ] c) Container runtime
- [ ] d) Image pulling

### 7. The right label set on a multi-tenant resource includes:
- [x] a) team, cost_center, environment, model_name (where applicable)
- [ ] b) Only namespace
- [ ] c) Only model_name
- [ ] d) Cluster name

### 8. Showback vs chargeback:
- [x] a) Showback shows cost; chargeback actually debits team budgets
- [ ] b) They're synonyms
- [ ] c) Showback is more rigorous than chargeback
- [ ] d) Chargeback comes before showback

### 9. PriorityClass enables:
- [x] a) Higher-priority pods preempting lower-priority pods under contention
- [ ] b) Faster scheduling for all pods
- [ ] c) Higher GPU bandwidth
- [ ] d) Lower image-pull latency

### 10. PodDisruptionBudget prevents:
- [x] a) Voluntary disruptions from reducing the number of healthy pods below a threshold
- [ ] b) Pod creation
- [ ] c) Image pulls
- [ ] d) Quota enforcement

### 11. The hardest part of cost attribution is usually:
- [x] a) Getting consistent labels on every resource
- [ ] b) Building the Grafana panel
- [ ] c) Choosing a cost API
- [ ] d) Pulling from kube-state-metrics

### 12. Hard multi-tenancy (separate clusters) is most justified for:
- [x] a) Regulated workloads with explicit isolation requirements
- [ ] b) Cost savings
- [ ] c) Simpler ops
- [ ] d) Smaller teams

### 13. Cross-team traffic should default to:
- [x] a) Denied; explicit NetworkPolicy required to allow
- [ ] b) Allowed; deny exceptions only
- [ ] c) Always allowed
- [ ] d) Routed through ingress

### 14. Fair-share queues are evaluated:
- [x] a) Continuously by the scheduler as resources free up
- [ ] b) Daily
- [ ] c) Only at submission time
- [ ] d) Hourly

### 15. The team that owns the platform should be billed for:
- [x] a) Platform overhead (control plane, ingress, monitoring) as an explicit line item
- [ ] b) Nothing
- [ ] c) All cluster cost
- [ ] d) Only their own pods

---

Answer key: 1.a 2.a 3.a 4.a 5.a 6.a 7.a 8.a 9.a 10.a 11.a 12.a 13.a 14.a 15.a
