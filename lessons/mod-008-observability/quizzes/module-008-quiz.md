# Module 08: Observability — Quiz

10 questions. 70% pass.

### 1. Generic HTTP metrics don't answer:
- [x] a) Is the model serving correctly (drift, accuracy)
- [ ] b) RPS
- [ ] c) Latency
- [ ] d) Error rate

### 2. PSI is used for:
- [x] a) Distributional drift detection
- [ ] b) Latency calculations
- [ ] c) Token counting
- [ ] d) Image quality

### 3. The right place to compute drift is:
- [x] a) A daily batch job comparing inference logs vs reference
- [ ] b) Inline in every prediction request
- [ ] c) Never; let the model degrade
- [ ] d) Manually by data scientists

### 4. A model owner's dashboard should answer (top to bottom):
- [x] a) Serving? → Latency? → Accuracy + drift? → Cost?
- [ ] b) Cost only
- [ ] c) Logs only
- [ ] d) GPU utilization only

### 5. Burn-rate alerts:
- [x] a) Trigger when error budget is being consumed faster than the allowed monthly rate
- [ ] b) Trigger on any error
- [ ] c) Replace SLOs
- [ ] d) Are unnecessary if you have logging

### 6. Multi-window burn-rate:
- [x] a) Two windows (e.g., 5m + 1h) both burning above threshold = page
- [ ] b) Single window only
- [ ] c) Average of all windows
- [ ] d) Required only for LLMs

### 7. Per-model cost attribution requires:
- [x] a) Required labels on every model resource (team, cost_center, model_name)
- [ ] b) Manual spreadsheets
- [ ] c) Per-pod billing
- [ ] d) PagerDuty integration

### 8. When PSI > 0.25 sustained:
- [x] a) Investigate; possibly retrain
- [ ] b) Ignore
- [ ] c) Scale up replicas
- [ ] d) Switch to a smaller model

### 9. Error budget policy:
- [x] a) Codifies when feature work pauses for reliability work
- [ ] b) Replaces SLOs
- [ ] c) Tracks cost
- [ ] d) Lists vendors

### 10. The most-missed signal in ML observability is:
- [x] a) Slice metrics (per-segment accuracy)
- [ ] b) Latency
- [ ] c) RPS
- [ ] d) Memory usage

---

Answers: all `a`
