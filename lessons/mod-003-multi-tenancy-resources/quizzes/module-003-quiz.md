# Module 03: Multi-Tenancy & Resource Management — Quiz

- 15 questions • 25 min • passing 75%

### 1. Default tenant isolation model for most ML platforms:
- [ ] a) Shared namespace with RBAC
- [x] b) Namespace-per-team
- [ ] c) Cluster-per-team
- [ ] d) Virtual cluster per team (vcluster)

### 2. ResourceQuota enforces at:
- [ ] a) Scheduling time only (after admission)
- [ ] b) Runtime via cgroup limits
- [x] c) Admission time (the apiserver rejects pods that exceed the quota)
- [ ] d) Build time during image creation

### 3. LimitRange in a namespace controls:
- [ ] a) Cluster-wide aggregate quotas
- [ ] b) NetworkPolicy ingress rules
- [x] c) Default + min/max per-container resource values
- [ ] d) RBAC subject bindings

### 4. The point of pairing default-deny NetworkPolicy with explicit allows:
- [ ] a) Deny-all alone blocks DNS so apps break
- [x] b) Default-deny is the secure baseline; allows codify only known traffic
- [ ] c) Required by the CNI plugin
- [ ] d) Improves throughput

### 5. Gang scheduling is needed when:
- [ ] a) Pods can each start independently
- [ ] b) A cron job runs daily
- [x] c) A distributed training job needs N pods running simultaneously or none at all
- [ ] d) The cluster is over-provisioned

### 6. Volcano / Yunikorn add to the default kube-scheduler:
- [ ] a) Node autoscaling
- [ ] b) Container runtime support
- [ ] c) Image-pull caching
- [x] d) Fair-share queues, gang scheduling, queue-based preemption

### 7. The minimum label set worth enforcing on every multi-tenant resource:
- [ ] a) `namespace` alone
- [ ] b) `model_name` alone
- [x] c) `team` + `cost_center` + `environment` + `model_name` where applicable
- [ ] d) Auto-generated cluster identifier

### 8. Showback vs chargeback:
- [x] a) Showback exposes cost to teams as information; chargeback actually debits team budgets
- [ ] b) They're synonyms; pick whichever the vendor uses
- [ ] c) Showback is more rigorous than chargeback
- [ ] d) Chargeback comes first; showback after a quarter

### 9. PriorityClass enables:
- [ ] a) Faster image pulls
- [ ] b) Higher GPU clock frequency
- [x] c) Higher-priority pods preempting lower-priority pods under contention
- [ ] d) Lower etcd write latency

### 10. PodDisruptionBudget prevents:
- [ ] a) Pods from being created
- [ ] b) Image pulls during deploys
- [x] c) Voluntary disruptions from reducing the number of healthy pods below a threshold
- [ ] d) ResourceQuota enforcement

### 11. The hardest part of cost attribution in practice is usually:
- [x] a) Getting consistent labels on every resource (otherwise the "unallocated" bucket grows)
- [ ] b) Building the Grafana panel
- [ ] c) Choosing between cloud cost APIs
- [ ] d) Pulling per-pod metrics from kube-state-metrics

### 12. Hard multi-tenancy (separate clusters per tenant) is most justified for:
- [ ] a) Cost savings
- [ ] b) Simpler day-to-day operations
- [x] c) Regulated workloads with explicit isolation requirements (HIPAA, FedRAMP, etc.)
- [ ] d) Smaller teams that don't share infra

### 13. Cross-team pod-to-pod traffic should default to:
- [ ] a) Allowed; exceptions are noted in NetworkPolicy
- [ ] b) Always allowed; trust the mesh
- [x] c) Denied; explicit NetworkPolicy required to allow
- [ ] d) Routed through the ingress gateway

### 14. Fair-share queue weights are evaluated:
- [ ] a) Once at job submission time
- [ ] b) Hourly
- [x] c) Continuously by the scheduler as resources free up
- [ ] d) Once per day

### 15. The platform team should be billed for:
- [ ] a) Nothing — they manage the cluster
- [ ] b) Every team's compute (since they own the infra)
- [ ] c) Only the pods their own services run
- [x] d) Platform overhead (control plane, ingress, monitoring) shown as an explicit line item on every team's bill

---

## Answer key + rationale

1. **b** — Namespace-per-team is the practical default. Shared namespace breaks team isolation; cluster-per-team is hard mode for ops; vcluster is overkill for most cases.
2. **c** — ResourceQuota is enforced by the apiserver's admission controller. Cgroups (runtime) handle LimitRange-style per-container limits, not aggregate quotas.
3. **c** — LimitRange sets per-container defaults + bounds; ResourceQuota sets aggregate caps.
4. **b** — Deny-all is the secure baseline; allows make the actual traffic pattern visible and reviewable.
5. **c** — Without gang scheduling, distributed training can have N-1 of N pods running while the Nth waits, wasting GPUs.
6. **d** — These are workload-scheduler features; node autoscaling + image pull are separate concerns.
7. **c** — Labels become the join key for cost rollups, policy enforcement, and observability slicing.
8. **a** — Showback is the introduction step; chargeback adds budget enforcement. Most teams stay on showback for months before chargeback to build trust.
9. **c** — Priority+preemption is exactly what differentiates the scheduler under contention.
10. **c** — PDB protects against *voluntary* disruptions (drain, upgrade). It does not stop OOM kills or node failures.
11. **a** — Labels are easy to forget on legacy workloads; the "unallocated" bucket grows until someone enforces via Kyverno.
12. **c** — Hard isolation is expensive; only worth it under explicit regulatory mandate.
13. **c** — Same logic as default-deny networking generally — explicit is safer than implicit.
14. **c** — Fair-share is a continuous re-evaluation; that's why it's superior to static quotas for bursty workloads.
15. **d** — Platform overhead must be allocated somewhere; an explicit line item beats hiding it in everyone's "unallocated" bucket.
