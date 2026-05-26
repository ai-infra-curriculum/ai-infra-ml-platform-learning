# Exercise 03: Resource Quota Management for Teams

## Objective

Design a complete *resource quota management system* for an ML platform's tenants. By the end you will produce a quota policy document, a worked spreadsheet (or table) of per-tier quotas, a cost-allocation model that ties quota usage to chargeback, a procedure for quota change requests, and an alerting design for quota exhaustion.

This exercise builds on Exercise 02. There, you wrote a single tenant's quota manifest. Here, you generalize: how does the *platform team* manage quotas across 30+ tenants over time?

## Learning Outcomes

By completing this exercise, you will:

- Design a tiered quota system that scales across many tenants.
- Apply fair-sharing principles to GPU allocation under contention.
- Build a cost-allocation model that gives tenants visibility into spend.
- Write a change-request procedure that respects auditability.
- Design alerting on quota exhaustion that's actionable, not noisy.

## Prerequisites

- Read Lecture 03 (Multi-Tenancy Patterns) in full, especially the sections on ResourceQuota, cost allocation, and noisy neighbors.
- Completed Exercise 02 (Namespace Isolation), or equivalent familiarity.
- Familiarity with Kubernetes ResourceQuota.
- Optional: familiarity with Kubecost, Kueue, or other Kubernetes cost/scheduling tools.

## Scenario

Same Aurelia AI scenario. You now have 12 tenants live on the platform; the org is projecting 30 by end of year. Last week:

- Tenant `ml-research` ran out of GPU quota at the worst possible time and missed a deadline. They want a quota increase.
- Tenant `ranking-team` is using 5% of their quota and has been for two months. They're sitting on capacity.
- The CFO emailed asking why cloud spend on the platform is up 35% month-over-month.
- A new tenant `voice-experiments` is being onboarded; they don't know what quota they need.

You realize: ad-hoc quota management is no longer working. You need a *system*.

## Deliverables

By the end of this exercise, you will have created:

1. `quota-policy.md` — the platform's quota policy document (what tiers exist, what each gets, how to request a change).
2. `quota-tiers.yaml` — the machine-readable tier definitions.
3. `cost-model.md` — the cost-allocation model and a worked example.
4. `change-request-procedure.md` — the workflow for quota change requests, including approval rules.
5. `alerting-design.md` — what alerts fire, to whom, at what thresholds.
6. `runbook-quota-exhaustion.md` — what an on-call engineer does when a tenant reports being out of quota.

---

## Part 1: Quota Tiers (25 minutes)

### Task 1.1: Define the tiers

A common pattern: define a small number of tiers (typically 3-5) with progressively higher limits. New tenants pick a tier; tenants can request promotion.

Sketch your tier table in `quota-policy.md`:

| Tier | CPU (req) | Memory (req) | GPUs | Pods | PVCs | Storage | Use case |
| --- | --- | --- | --- | --- | --- | --- | --- |
| **starter** | 10 | 64Gi | 0 | 30 | 10 | 200Gi | New tenants, exploration, no GPU work |
| **standard** | 40 | 256Gi | 4 | 100 | 50 | 2Ti | Active ML team doing modest training |
| **premium** | 100 | 1Ti | 16 | 300 | 100 | 10Ti | Heavy production training workloads |
| **mission-critical** | 500 | 4Ti | 64 | 1000 | 500 | 50Ti | Critical large-scale workloads with VP sign-off |

**TODO**: Fill in your own version. Some questions to think about:

- Are these numbers right for Aurelia (Series B, 30 ML engineers, 6 teams)? If you imagine those 30 engineers split into 6 teams, what tier does each typically need?
- Is the gap between standard and premium too big? Many tenants will want "more than standard but less than premium." Is that a real need or are they just hoarding?
- The mission-critical tier has VP sign-off. Why? What does that gate prevent?

### Task 1.2: Express tiers as YAML

For machine-readability and GitOps:

```yaml
# quota-tiers.yaml
apiVersion: platform.aurelia.example.com/v1
kind: QuotaTierCatalog
metadata:
  name: tier-catalog-2026Q2
spec:
  tiers:
    - name: starter
      description: "New tenants, exploration, no GPU work."
      quota:
        requests.cpu: "10"
        requests.memory: "64Gi"
        requests.nvidia.com/gpu: "0"
        pods: "30"
        persistentvolumeclaims: "10"
        requests.storage: "200Gi"
      limits:
        # Per-pod LimitRange caps
        max.cpu: "8"
        max.memory: "32Gi"
      cost_estimate_usd_per_month: 850
      approval_required: false

    - name: standard
      description: "Active ML team doing modest training."
      quota:
        requests.cpu: "40"
        requests.memory: "256Gi"
        requests.nvidia.com/gpu: "4"
        pods: "100"
        persistentvolumeclaims: "50"
        requests.storage: "2Ti"
      limits:
        max.cpu: "16"
        max.memory: "64Gi"
      cost_estimate_usd_per_month: 8200
      approval_required: false

    # TODO: fill in premium and mission-critical
```

**TODO**: Complete `quota-tiers.yaml`. Note that cost estimates depend on cloud pricing; the numbers above are illustrative. Don't agonize over the exact numbers.

### Task 1.3: Default tier and self-service

What tier does a brand-new tenant get without approval?

Most platforms set this to `starter` (or equivalent low tier) and require approval for higher tiers. The reasoning: cost-bounded blast radius for newcomers, plus it forces a conversation about what they actually need.

**TODO**: Write a paragraph in `quota-policy.md` covering:

- The default tier for new tenants.
- Whether tenants can self-promote within the catalog (e.g., starter → standard) or must request.
- The maximum tier a tenant can have without VP sign-off.

### Task 1.4: Tier inflation

Once a system exists, every tenant wants to be on the highest tier. This is *tier inflation*. Without intervention, after a year you have 30 tenants all on `mission-critical` and no actual contention prevention.

Mitigations to consider:

- **Charge real money.** Tenants on a higher tier pay more (showback or chargeback). Most tenants will self-select down.
- **Periodic review.** Every quarter, tenants justify their tier. Those that haven't used >50% of their tier get downgraded.
- **Approval friction.** Tier upgrades go through a defined approval process with delay; tenants who don't truly need higher tier won't bother.
- **Burst capacity.** Tenants get a *base* quota (their tier) plus *burst* capacity for short-term needs without permanent upgrade.

**TODO**: Pick at least two mitigations and document them in `quota-policy.md`.

---

## Part 2: Cost Allocation Model (25 minutes)

### Task 2.1: Cost-per-resource

Aurelia's cloud pricing (illustrative; not real Aurelia prices):

- General-purpose CPU: $0.04 per core-hour
- General-purpose memory: $0.006 per GB-hour
- A100 GPU: $3.40 per GPU-hour
- H100 GPU: $4.50 per GPU-hour
- Persistent storage (gp3): $0.000130 per GB-hour ($0.10 per GB-month)
- Object storage (S3 standard): $0.000031 per GB-hour ($0.023 per GB-month)
- Egress to internet: $0.09 per GB

For each tier, compute the *worst-case monthly cost* (if the tenant uses 100% of quota 100% of the time):

```
standard tier:
  CPU:    40 cores × $0.04/h × 730 h/month       = $1,168
  Memory: 256GB × $0.006/h × 730 h/month         = $1,121
  GPUs:    4 × $3.40/h × 730 h/month             = $9,928
  Storage: 2,000 GB × $0.000130/h × 730 h/month  =   $190
  TOTAL (max)                                    = $12,407 / month
```

**TODO**: Compute the same for `starter`, `premium`, and `mission-critical`. Put the numbers in `cost-model.md`. Be explicit about the assumptions ("100% utilization for the whole month is the worst case; typical utilization is X%").

### Task 2.2: Typical utilization

100% utilization is unrealistic. Most tenants use perhaps 20-40% of their quota over a month. Build a *typical-case* estimate:

| Tier | Worst-case $/mo | Typical (30%) $/mo |
| --- | --- | --- |
| starter | $850 | $255 |
| standard | $12,407 | $3,722 |
| premium | ~$X | ~$Y |
| mission-critical | ~$X | ~$Y |

**TODO**: Compute the typical-case row for each tier.

### Task 2.3: Showback dashboard

A tenant should be able to see their cost. Design a simple dashboard with these panels:

- **This-month-to-date spend.** Total cost incurred so far this calendar month.
- **Spend by resource type.** CPU vs memory vs GPU vs storage vs egress — which is dominant?
- **Quota utilization.** % of each quota dimension currently committed.
- **Top 5 most expensive workloads.** Which specific pods/jobs cost the most?
- **Month-over-month trend.** Is spend going up or down?
- **Forecast for end of month.** Linear projection.

**TODO**: Sketch the dashboard in `cost-model.md` as ASCII or as a description. For each panel, note the data source (Kubecost, Prometheus, custom metrics, billing API).

### Task 2.4: Chargeback rules

Eventually showback becomes chargeback. Define the chargeback rules:

- **What costs are tenant-owned?** Their CPU/memory/GPU/storage usage, plus a per-tenant fixed share of platform overhead.
- **What costs are platform-team-owned?** Control-plane services (registry, training orchestrator, monitoring stack), shared infrastructure (NAT gateways, load balancers, etc.).
- **Granularity.** Monthly invoicing? Daily? Real-time?
- **Reconciliation cycle.** When does the platform "close the books" for a month? What happens to costs that come in late?

**TODO**: Write a paragraph in `cost-model.md` covering each of these.

### Task 2.5: A worked example

In `cost-model.md`, work through a concrete example: tenant `ranking-team` used the following resources last month:

- Average CPU committed: 12 cores
- Average memory committed: 80 GB
- GPU-hours used: 320 (4 GPUs × 80 hours)
- Persistent storage: 400 GB for the full month
- Egress: 50 GB

Compute their chargeback:

```
CPU:     12 × $0.04 × 730                         = $350.40
Memory:  80 × $0.006 × 730                        = $350.40
GPU:     320 × $3.40                              = $1,088
Storage: 400 × $0.10                              = $40
Egress:  50 × $0.09                               = $4.50
Subtotal                                          = $1,833.30

Platform overhead allocation (1/12 of monthly platform cost,
  amortized evenly across 12 tenants):
  Platform monthly cost: $24,000
  Per-tenant share:      $2,000

TOTAL                                             = $3,833.30
```

**TODO**: Walk through this calculation. Note where you have to make a judgment call (the platform overhead allocation method, in particular).

---

## Part 3: Quota Change Request Procedure (15 minutes)

### Task 3.1: Define the procedure

When a tenant needs a quota change, what happens? Document the flow in `change-request-procedure.md`:

1. **Submission.** Tenant submits a request via a form/CLI/API. Required fields:
   - Tenant name
   - Requested change (which dimension, by how much, permanent or temporary)
   - Justification (free text + structured fields like "expected GPU-hours per week," "duration if temporary")
   - Cost-center approval (signed off by tenant's manager)
   - Expected duration (permanent or end date)
2. **Validation.** The platform automatically checks:
   - Is the tenant currently using >70% of their existing quota? If <50%, auto-deny with a friendly message.
   - Does the requested quota exceed the tier the tenant is on? If yes, route to tier-upgrade flow.
   - Is the request within reasonable bounds (e.g., not asking for 1000 GPUs)?
3. **Approval.**
   - Within-tier increases: platform-team approver clicks approve.
   - Tier upgrades: platform-team approver + tenant's VP sign off.
   - Temporary increases (< 7 days): platform-team approver only, auto-expire.
4. **Application.** Approved changes are merged to the GitOps repo. CD applies them.
5. **Notification.** Tenant is notified that the change is live. Cost dashboard updates.
6. **Audit.** Every change is logged with the requester, approver, justification, before/after values, and timestamps.

**TODO**: Write this procedure in `change-request-procedure.md`. Add timing expectations:

- How long should approval take? (Typically: ≤ 1 business day for within-tier; ≤ 5 business days for tier upgrades.)
- What happens during off-hours? (On-call platform engineer can approve temporary increases for genuine production emergencies.)

### Task 3.2: Define rejection criteria

When does the platform team *reject* a quota request?

- Tenant is using <50% of their existing quota in that dimension. (They don't need more; they have more.)
- Requested increase would put them over their tier's cap. (Need tier upgrade instead.)
- No cost-center approval. (Procedural; the cost has to be authorized.)
- Justification is missing or boilerplate ("we need more").
- Requested duration is permanent for what is clearly a temporary need.
- Tenant has unresolved billing issues.

**TODO**: Write these in `change-request-procedure.md`. For each, write a template response message the platform team can copy-paste.

### Task 3.3: The "temporary increase" pattern

For a one-week sprint where a tenant genuinely needs 2x normal GPU, you don't want to permanently inflate their tier. Use *temporary increases*:

```yaml
apiVersion: platform.aurelia.example.com/v1
kind: TemporaryQuotaIncrease
metadata:
  name: ml-research-april-sprint
spec:
  tenant: ml-research
  reason: "Q2 sprint for model X retraining"
  increases:
    requests.nvidia.com/gpu: "+4"   # temporary: 4 → 8
  expires_at: "2026-05-08T00:00:00Z"
  approved_by: platform-admin
  cost_center_approver: ml-research-lead
```

The platform reconciles this CR into the actual ResourceQuota, then automatically rolls back when `expires_at` passes.

**TODO**: Document this pattern in `change-request-procedure.md`. Note: implementing this requires platform code (a controller). For the purposes of this exercise, the design is enough.

---

## Part 4: Fair Sharing under Contention (15 minutes)

ResourceQuota caps per-tenant *commitment*, but it does not handle *contention*. When 8 GPUs are physically available and 3 tenants each want 4, who gets which?

### Task 4.1: Decide on a scheduling policy

The choices (from Lecture 03):

- FIFO
- Round-robin
- Fair-share (DRF or similar)
- Weighted fair-share
- Priority + fair-share

For an internal ML platform with a mix of production and exploration workloads, **priority + fair-share** is usually the right answer.

**TODO**: In `quota-policy.md`, write a section "Scheduling Policy" covering:

- The priority classes (e.g., `production-critical`, `production-standard`, `experiment`, `low-priority`)
- The fairness model within a priority
- How weights are assigned (per-tenant)
- Whether preemption is enabled

Example draft:

```
We use Kueue with the following priority classes:

| Priority             | Weight | Preemption |
| -------------------- | ------ | ---------- |
| production-critical  | 1000   | Cannot be preempted |
| production-standard  | 100    | Can preempt experiment / low-priority |
| experiment           | 10     | Can be preempted by production |
| low-priority         | 1      | Always preemptable |

Within a priority, fair-share by tenant. Each tenant's weight equals
the number of FTE engineers on the team (so a 6-person team gets 3x
the GPU access of a 2-person team during contention).
```

### Task 4.2: Backfilling

When GPU 1 is allocated and GPU 2-8 are idle, the scheduler should *backfill* idle GPUs with lower-priority jobs even if those jobs would normally wait. Document this in `quota-policy.md`.

### Task 4.3: The queue UX

When a tenant submits and there's contention, what do they see?

Bad: "queued." (No information.)

Good: "queued; 4 jobs ahead in your priority + tenant; estimated start: 23 minutes."

**TODO**: Note in `quota-policy.md` that the platform must expose queue position and ETA in the API.

---

## Part 5: Alerting Design (15 minutes)

Alerts on quota events tell the platform team and the tenant when something needs attention. Design them in `alerting-design.md`.

### Task 5.1: Tenant-facing alerts

Alerts to the tenant (delivered via email, Slack, or in-platform notification):

1. **Quota approaching exhaustion.** When a tenant uses >85% of their quota in any dimension, notify them. Once per day, max.
2. **Quota fully exhausted, new requests rejected.** When a tenant hits 100% and a request is rejected, notify them immediately with a link to request more.
3. **Temporary quota increase expiring soon.** 3 days before a temporary increase expires, remind the tenant.
4. **Workload abandoned.** If a tenant's pod has been Pending for >1 hour due to quota, notify them.

For each, define:
- The trigger (precise metric + threshold)
- The delivery channel
- The frequency limit (avoid spam)
- The action the tenant should take

**TODO**: Write each alert in `alerting-design.md`.

### Task 5.2: Platform-team-facing alerts

Alerts to the platform team:

1. **Tenant submitted a quota change request.** Route to platform-team Slack channel.
2. **Tenant has been at >95% quota for >7 days.** Suggest a quota review.
3. **Tenant cost is up >50% week-over-week.** Investigate anomaly.
4. **Cluster-wide GPU utilization is low (<30%) AND tenants are queueing.** Indicates a bin-packing or fairness bug; the scheduler isn't matching demand to supply.
5. **A temporary quota increase has been active longer than 30 days.** Should it become permanent?

**TODO**: Write each alert in `alerting-design.md`.

### Task 5.3: Anti-noise rules

Common alert-fatigue traps:

- Don't alert on transient utilization spikes — require sustained >85% for ≥1 hour.
- Don't alert daily for the same condition — once per condition per day max.
- Allow tenants to snooze alerts for 24 hours.
- Make every alert actionable. If the recipient can't do anything about it, don't alert.

**TODO**: Document these rules in `alerting-design.md`.

---

## Part 6: On-Call Runbook (10 minutes)

When a tenant pages the platform team at 11pm because they're out of quota and have a deadline, what does the on-call engineer do?

Create `runbook-quota-exhaustion.md`:

```markdown
# Runbook: Tenant Reports Quota Exhaustion

## Symptom
A tenant reports that their pod/job cannot be created. The error message
contains "exceeded quota" or "forbidden: pods quota". They are requesting
immediate intervention.

## Triage (5 minutes)

1. **Identify the tenant.** Get the tenant name from the requester.

2. **Check current usage.**
   ```bash
   kubectl get resourcequota -n tenant-${TENANT} -o yaml
   ```
   Confirm which dimension(s) are exhausted.

3. **Check the workload they're trying to submit.**
   Ask them to share the failing pod/job manifest. Verify the request size
   is reasonable. (If they're trying to submit a 50-GPU job, the quota
   isn't the real problem.)

4. **Determine urgency.**
   - Production-critical: address immediately.
   - Exploration: address in business hours.

## Mitigation Options

In rough order of preference:

### Option A: Tenant clears stuck/idle workloads

Ask the tenant to identify and delete pods/jobs they no longer need.
This is the most common resolution; tenants often have abandoned pods
consuming quota.

### Option B: Apply a temporary quota increase

For genuine production needs, apply a TemporaryQuotaIncrease (see
change-request-procedure.md). On-call has authority to approve
temporary increases of ≤2x current quota for ≤72 hours without
further sign-off.

   ```bash
   kubectl apply -f - <<EOF
   apiVersion: platform.aurelia.example.com/v1
   kind: TemporaryQuotaIncrease
   metadata:
     name: ${TENANT}-oncall-$(date +%Y%m%d)
   spec:
     tenant: ${TENANT}
     reason: "${REASON}"
     increases:
       ${DIMENSION}: "+${AMOUNT}"
     expires_at: "$(date -u -d '+72 hours' +%Y-%m-%dT%H:%M:%SZ)"
     approved_by: ${YOUR_USERNAME}
   EOF
   ```

### Option C: Reduce the tenant's workload

If the tenant has a single very large pod, suggest they decompose
it into smaller pods. (This is a longer conversation; not a 2am fix.)

### Option D: Escalate

If the tenant insists on a permanent quota increase and you can't
approve it, escalate to the platform-team lead. Do NOT permanently
increase quota at 2am without a second pair of eyes.

## Post-Incident

- Log the incident in the platform incident channel.
- File a follow-up ticket for review during the next business day:
  - Why did the tenant hit quota?
  - Was the response time acceptable?
  - Was the temporary increase appropriate?
- If the tenant frequently hits quota, propose a tier review.

## Anti-patterns (do not do)

- Increasing the quota permanently at 2am without approval.
- Granting the tenant cluster-admin to bypass quota. (No.)
- Deleting another tenant's workloads to free room. (Hard no.)
```

**TODO**: Adapt this runbook to your scenario. If you have additional steps specific to your design (e.g., a Kueue command, a Slack runbook bot), include them.

---

## Part 7: Reflection and Critique (10 minutes)

### Task 7.1: Stress-test your design

Walk through these scenarios and check that your design handles them:

1. **Sudden surge.** Three tenants simultaneously want 100% of their GPU quota for an urgent retraining. What happens? (Hint: contention scheduler + priority classes.)
2. **Slow leak.** Tenant `unused-team` has 4 GPU quota and uses 0.2 GPUs on average. They've been like this for 6 months. What is your system supposed to do? (Hint: periodic review + downgrade.)
3. **Abuse attempt.** A tenant continually requests temporary increases, each for "just 24 hours," and they accumulate. Does your design prevent this? (Hint: temporary-increase tracking + rolling-window limits.)
4. **The CFO's request.** The CFO emails: "Cut cloud spend by 20% in 30 days." What do you do, given your design? (Hint: showback dashboard surfaces idle quota; downgrade unused tenants; raise prices.)
5. **Cluster-wide emergency.** A major incident requires the platform team to free up 50% of cluster capacity for 6 hours. How do you accomplish this without breaking tenant trust? (Hint: pre-defined emergency posture; communicate broadly; prioritize critical workloads.)

**TODO**: For each scenario, write 2-4 sentences in `quota-policy.md` under a "Stress Tests" section, explaining how your design handles it (or doesn't).

### Task 7.2: Self-critique

What are the weaknesses of your design? At least three. Examples:

- "The tier catalog is static. As real-world demand changes, the tiers will get out of date."
- "Cost allocation depends on accurate labels, which depends on tenant discipline. A tenant who forgets to label gets free cost."
- "The temporary-increase flow has no rate limit on how often a tenant can ask. They could pile up daily requests."

**TODO**: Write three self-critiques in `quota-policy.md`.

---

## Common Pitfalls

1. **One-size-fits-all quotas.** Treating a 2-person team and a 30-person team identically is wasteful for one and starvation for the other.
2. **No mechanism to reclaim unused quota.** Without reclamation, tier inflation leads to a permanent over-provisioning state.
3. **Charging too early or too late.** Charging from day one alienates new tenants; never charging eliminates the cost discipline.
4. **Burdensome change requests.** If asking for more quota requires a 2-week approval chain, tenants will route around it (e.g., by spinning up shadow infrastructure in another account).
5. **Alerting on every minor event.** Tenants will mute the channel, and real alerts get lost.
6. **No on-call story for quota issues.** Even with auto-scaling and burst capacity, sometimes a human has to make a judgment call at 11pm.

---

## Reflection Questions

In `quota-policy.md` under a "Reflection" heading, answer:

1. The default-tier choice (starter vs standard) for new tenants is a values choice. What does choosing `starter` say about your platform's philosophy? What does choosing `standard` say?
2. Your fair-sharing scheduler weights each tenant by team size. What's the implicit assumption here? When would that assumption be wrong?
3. You set the platform overhead allocation to "evenly across tenants." A small tenant pays the same overhead share as a large one. Is that fair? What's the alternative?
4. You expose cost in real time to tenants. Does this incentivize good behavior or *gaming*? (E.g., tenants might split workloads across labels to hide cost.)
5. Imagine you propose this whole system to Aurelia's VP of Engineering. What pushback do you expect? How do you respond?

---

## Self-Assessment

- [ ] Can I name the tiers and their distinguishing features?
- [ ] Can I compute the worst-case monthly cost of one tier from memory?
- [ ] Can I walk through the change-request flow on a whiteboard?
- [ ] Can I list at least 4 alerts and what triggers each?
- [ ] If I were the on-call engineer at 2am, do I know what to do?

If yes to all, you're done.

---

## Suggested Time Allocation

| Section | Time |
| --- | --- |
| Part 1: Quota Tiers | 25 min |
| Part 2: Cost Allocation Model | 25 min |
| Part 3: Change Request Procedure | 15 min |
| Part 4: Fair Sharing | 15 min |
| Part 5: Alerting Design | 15 min |
| Part 6: On-Call Runbook | 10 min |
| Part 7: Reflection and Critique | 10 min |
| **Total** | **115 min** |

---

## Where to Go from Here

You now have an end-to-end quota management design. In Modules 03 (Kubernetes) and 04 (Training Infrastructure) you'll see the runtime side — how Kueue, Volcano, or other schedulers actually implement fair-sharing. Module 09 (Security) revisits the auditability of these changes. Module 10 (Governance) revisits cost as a governance lever.

For now, push your deliverables to the curriculum repo fork, and move on to Exercise 04.

---

## Appendix A: Worked Example — A Full Quarterly Quota Review

If you have time, work through this concrete simulation as an extension to the exercise. It's the kind of analysis you would do every quarter.

### The scenario

At the start of Q3, you (the platform team) sit down to review the previous quarter's tenant usage. You pull a CSV of average usage per tenant over the last 90 days:

| Tenant | Tier | CPU avg | CPU quota | Mem avg | Mem quota | GPU avg | GPU quota | $ spent |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| ranking-team | standard | 12 / 40 | 30% | 80 / 256 | 31% | 0.3 / 4 | 8% | $4,200 |
| ml-research | standard | 35 / 40 | 88% | 220 / 256 | 86% | 3.8 / 4 | 95% | $18,400 |
| voice-experiments | starter | 6 / 10 | 60% | 40 / 64 | 63% | 0 / 0 | n/a | $510 |
| churn-team | standard | 8 / 40 | 20% | 60 / 256 | 23% | 0.5 / 4 | 13% | $3,600 |
| recsys-team | premium | 65 / 100 | 65% | 700 / 1024 | 68% | 12 / 16 | 75% | $48,000 |
| forecasting | starter | 3 / 10 | 30% | 25 / 64 | 39% | 0 / 0 | n/a | $290 |
| safety-eval | standard | 22 / 40 | 55% | 150 / 256 | 59% | 2 / 4 | 50% | $9,200 |
| analytics-ml | standard | 14 / 40 | 35% | 90 / 256 | 35% | 0.1 / 4 | 3% | $3,800 |

### Task A.1: Triage

For each tenant, decide one of: **upgrade-candidate**, **downgrade-candidate**, **status-quo**, **investigate**.

- *Upgrade-candidate*: utilization > 80% in any dimension AND the tenant has not had a tier change in 90 days.
- *Downgrade-candidate*: utilization < 25% across all dimensions for 90+ days.
- *Investigate*: utilization mismatch (e.g., high in one dimension, near-zero in another) that suggests a config bug.
- *Status-quo*: neither of the above.

**TODO**: Triage each tenant. (Hint: ml-research is upgrade-candidate; ranking-team and analytics-ml and churn-team are downgrade-candidates; recsys-team is status-quo; etc.)

### Task A.2: Recommended actions

For each candidate, write a one-line action:

- ml-research: propose tier upgrade to premium, with a meeting to discuss workload patterns.
- ranking-team: propose downgrade to starter, with a friendly note explaining the data.
- analytics-ml: investigate why GPU is at 3% — is the team waiting on something? Is the quota wasted?
- churn-team: similar to ranking-team — downgrade to starter.

**TODO**: Write the actions in your `quota-policy.md` under an "Appendix A: Q2 Review" section.

### Task A.3: Communication templates

The hardest part of quota management is the *conversations*. Write templated messages for:

- **The upgrade conversation.** "Hi ml-research, your usage data shows you've been at 86% memory utilization and 95% GPU utilization. We'd like to propose upgrading you to the premium tier. Here's what changes: ..."
- **The downgrade conversation.** This is more delicate. "Hi ranking-team, we noticed your usage has been at 20-30% of your standard tier quota for the last quarter. We'd like to propose moving you to the starter tier, which would reduce your monthly cost from $3,600 to $850. Would you like to discuss?"
- **The investigation conversation.** "Hi analytics-ml, we noticed your GPU quota is at 4 but you're only using 0.1 on average. Is there a workload you've been waiting on? Or should we reclaim that capacity?"

**TODO**: Write these templates. Soften the downgrade message; nobody likes losing capacity.

### Task A.4: Roll up to the CFO

The CFO asked why cloud spend was up 35%. Summarize what your review showed:

- Total platform spend last quarter: $88,000 (sum of $ spent column).
- Three tenants are over-tiered ($11,600 of quota effectively unused on 3 tenants).
- One tenant is under-tiered (ml-research is queueing work; productivity impact is hard to quantify).
- The spend growth was driven by recsys-team (premium, $48,000) and ml-research (high utilization, $18,400). Other tenants are flat or down.

**TODO**: Write a 5-bullet executive summary you'd send the CFO. Cover: total spend, key drivers, recommended actions, savings opportunities, expected next-quarter spend.

### Task A.5: Reflection

How did this exercise feel? What was surprising? What information would you wish your platform produced automatically to make this review easier?

**TODO**: Write 2-3 sentences in your `quota-policy.md`.

---

## Appendix B: Quota Anti-patterns (Quick Reference)

For your future reference, the most common quota-management anti-patterns:

1. **The Christmas Tree.** Every tenant gets exactly the same quota regardless of size. Wastes capacity for small teams; starves large teams.
2. **The Squeaky Wheel.** Whichever team complains loudest gets a quota bump. Quota becomes a function of political effectiveness, not need.
3. **The Quota Bazaar.** Quota changes are negotiated case-by-case with no codified policy. Tenants compete with each other for limited capacity; platform team becomes a referee.
4. **The Frozen Quota.** Quotas are set once and never revisited. Tenants either chronically starve or chronically waste.
5. **The Invisible Quota.** Tenants don't know what their quota is until they hit it. Surprise rejection erodes trust.
6. **The Free Lunch.** Quotas exist but cost is not visible to tenants. They have no reason to use less.
7. **The Punitive Quota.** Quota changes are framed as discipline ("your team uses too much"). Tenants hide their real needs.

The healthy mode looks like: codified tiers, transparent cost, periodic review, easy upgrade path, visible utilization, downgrade by data, upgrade by need. None of those are difficult individually; the discipline is in doing all of them consistently.
