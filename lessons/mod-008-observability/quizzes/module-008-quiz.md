# Module 08: Observability — Quiz

10 questions. 70% pass.

### 1. Generic HTTP metrics (RPS, latency, error rate) DON'T answer:
- [ ] a) How many requests per second
- [ ] b) p95 response latency
- [ ] c) Server error rate
- [x] d) Whether the model is serving *correctly* (drift, accuracy regression)

### 2. PSI (Population Stability Index) is used for:
- [ ] a) Latency percentile calculations
- [ ] b) Token counting in LLM inference
- [ ] c) Image quality scoring
- [x] d) Distributional drift detection between two windows

### 3. The right place to compute drift is:
- [x] a) A daily batch job comparing recent inference logs vs a reference window
- [ ] b) Inline on every prediction request
- [ ] c) Never — let the model degrade until users complain
- [ ] d) Manually by data scientists in notebooks

### 4. A model owner's dashboard should answer top-to-bottom:
- [ ] a) Cost only
- [ ] b) Logs only
- [ ] c) GPU utilization only
- [x] d) Is it serving? → How well (latency)? → Is it correct (accuracy + drift)? → What does it cost?

### 5. Burn-rate alerts fire when:
- [ ] a) Any single error occurs
- [ ] b) The SLO is missed entirely
- [ ] c) The logging pipeline backs up
- [x] d) Error budget is being consumed faster than the allowed monthly burn rate

### 6. Multi-window burn-rate alerts use:
- [x] a) Two windows (e.g., 5m + 1h) both burning above threshold → page
- [ ] b) A single window's instantaneous rate
- [ ] c) Average burn over all open windows
- [ ] d) Are LLM-specific and don't apply to other ML

### 7. Per-model cost attribution requires:
- [x] a) Required labels (`team`, `cost_center`, `model_name`) on every model resource
- [ ] b) Manual spreadsheets reconciled monthly
- [ ] c) A per-pod billing relationship with the cloud provider
- [ ] d) A PagerDuty integration

### 8. When PSI > 0.25 sustains for an extended window:
- [x] a) Investigate; the input distribution has materially shifted — possibly retrain
- [ ] b) Ignore — PSI fluctuates naturally
- [ ] c) Scale up serving replicas
- [ ] d) Switch the model to a smaller variant

### 9. An error budget policy is:
- [ ] a) A replacement for SLOs
- [ ] b) A cost tracker
- [ ] c) A vendor selection checklist
- [x] d) A codified rule for when feature work pauses to make room for reliability work

### 10. The most-missed signal in ML observability (vs generic web obs) is:
- [x] a) Slice metrics — accuracy on each meaningful customer segment
- [ ] b) Latency
- [ ] c) Request-per-second
- [ ] d) Memory usage

---

## Answer key + rationale

1. **d** — RPS/latency/errors tell you the *system* is responding; they say nothing about whether the *model* is doing the right thing.
2. **d** — PSI compares two distributions; standard drift metric.
3. **a** — Inline drift would add latency to every request; daily batch is cheap + sufficient.
4. **d** — The hierarchy mirrors a debugging walkthrough: connectivity → speed → correctness → economics.
5. **d** — Burn-rate is specifically about budget exhaustion velocity.
6. **a** — Two-window-both-firing reduces false positives from instantaneous spikes.
7. **a** — Labels are how cost rolls up to the right team; without them, "unallocated" balloons.
8. **a** — 0.25 is the conventional "significant shift" threshold; sustained shift warrants action.
9. **d** — The policy is what makes budgets real — without it teams ignore SLOs.
10. **a** — Aggregate metrics hide per-segment regressions; a fairness-relevant slice can collapse silently.
