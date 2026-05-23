# Module 09: Security & Governance — Quiz

10 questions. 70% pass.

### 1. AuthN vs AuthZ:
- [x] a) AuthN = who; AuthZ = what they can do
- [ ] b) Synonyms
- [ ] c) AuthZ comes before AuthN
- [ ] d) Only AuthN matters

### 2. The best mechanism for pod → cloud service auth:
- [x] a) Workload identity (IRSA / GCP WI)
- [ ] b) Static IAM access keys in env vars
- [ ] c) Shared secret in ConfigMap
- [ ] d) IP allowlist

### 3. OIDC is most appropriate for:
- [x] a) Human user AuthN
- [ ] b) Pod-to-pod traffic
- [ ] c) Service-to-cloud
- [ ] d) Image signing

### 4. SLSA L2 requires:
- [x] a) Signed provenance from a hosted build service
- [ ] b) Open-source-only deps
- [ ] c) On-prem build
- [ ] d) Zero CVEs

### 5. Cosign keyless signing uses:
- [x] a) Sigstore + OIDC identity at sign time
- [ ] b) Pre-shared keys
- [ ] c) RSA keys committed to git
- [ ] d) HSM-only

### 6. Image admission verification in Kubernetes:
- [x] a) Kyverno or Sigstore Policy Controller checks signatures at admission
- [ ] b) Manual review
- [ ] c) Image pull policy
- [ ] d) ImageStreams (OpenShift only)

### 7. Governance gates at promotion typically include:
- [x] a) Model card + bias review + decision log + audit entry
- [ ] b) GPU benchmark only
- [ ] c) Vault token check only
- [ ] d) Latency test

### 8. EU AI Act low-risk tier compliance typically requires:
- [x] a) Model cards + bias reviews + audit trail
- [ ] b) Full lifecycle simulation
- [ ] c) Open-sourcing the model
- [ ] d) GPU verification

### 9. The audit log should be:
- [x] a) Append-only with hash chaining (tamper-evident)
- [ ] b) Rewritable
- [ ] c) Stored in plaintext
- [ ] d) Per-day rotated

### 10. Model artifacts in the supply chain:
- [x] a) Are also security-relevant; use safe formats + sign them
- [ ] b) Don't matter
- [ ] c) Only matter at training
- [ ] d) Excluded from SLSA

---

Answers: all `a`
