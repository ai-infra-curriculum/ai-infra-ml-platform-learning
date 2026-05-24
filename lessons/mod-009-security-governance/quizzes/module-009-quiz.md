# Module 09: Security & Governance — Quiz

10 questions. 70% pass.

### 1. AuthN vs AuthZ — the distinction is:
- [ ] a) They are synonyms; use whichever your stack prefers
- [ ] b) AuthZ runs before AuthN in the request path
- [ ] c) AuthN is for humans, AuthZ is for services
- [x] d) AuthN proves *who* the caller is; AuthZ decides *what* they can do

### 2. The recommended mechanism for pod → cloud service authentication is:
- [x] a) Workload identity (IRSA on EKS, GCP WI, Azure AD pod identity)
- [ ] b) Static IAM access keys baked into env vars
- [ ] c) A shared secret in a ConfigMap
- [ ] d) IP allowlist on the cloud-service side

### 3. OIDC is most appropriate for:
- [x] a) Human-user authentication (SSO into platform APIs/consoles)
- [ ] b) Pod-to-pod traffic inside the mesh
- [ ] c) Service-to-cloud-service auth
- [ ] d) Container image signing

### 4. SLSA Level 2 requires (minimum):
- [x] a) Signed provenance produced by a hosted (not local) build service
- [ ] b) All dependencies must be open source
- [ ] c) Builds must run on-premises
- [ ] d) Zero CVEs in the resulting image

### 5. Cosign keyless signing relies on:
- [x] a) Sigstore + an OIDC identity asserted at sign time (no static keys)
- [ ] b) Pre-shared symmetric keys
- [ ] c) RSA keys committed to the git repo
- [ ] d) An HSM exclusively

### 6. Image signature verification in Kubernetes happens via:
- [x] a) Kyverno or Sigstore Policy Controller as an admission webhook
- [ ] b) Manual code review before kubectl apply
- [ ] c) The pod's image pull policy
- [ ] d) OpenShift ImageStreams (only)

### 7. Governance gates at Production promotion typically include:
- [x] a) Model card + bias review + decision log + audit log entry
- [ ] b) GPU performance benchmark
- [ ] c) A Vault token validation step only
- [ ] d) A latency regression test only

### 8. EU AI Act low-risk-tier compliance typically requires:
- [x] a) Model cards + bias reviews + audit trail
- [ ] b) Full lifecycle simulation before each release
- [ ] c) Open-sourcing the model weights
- [ ] d) GPU hardware verification

### 9. The audit log should be:
- [x] a) Append-only with hash chaining (tamper-evident)
- [ ] b) Rewritable so corrections are easy
- [ ] c) Stored in plaintext for grep-ability
- [ ] d) Rotated daily and prior days deleted

### 10. Model artifacts (weights, adapters) in the supply chain:
- [x] a) Are security-relevant; use safe formats + sign them like images
- [ ] b) Don't matter for security; only code matters
- [ ] c) Only matter during training, not serving
- [ ] d) Are explicitly out of scope for SLSA

---

## Answer key + rationale

1. **d** — AuthN = identity; AuthZ = permission. Conflating them is a common security bug.
2. **a** — Static keys leak; workload identity eliminates the credential entirely.
3. **a** — OIDC is built for human SSO flows; pod-to-pod uses mTLS, service-to-cloud uses workload identity.
4. **a** — L2's distinguishing requirement is hosted build + signed provenance.
5. **a** — Keyless = identity is the OIDC assertion, not a static key file.
6. **a** — Admission-time enforcement; pull policy only controls *whether* to pull, not *trust*.
7. **a** — These four artifacts are the standard governance gate set for regulated ML.
8. **a** — Low-risk tier focuses on transparency + bias controls + auditability.
9. **a** — Tamper-evident audit logs are required for any meaningful incident-response or compliance story.
10. **a** — A compromised model artifact is as dangerous as a compromised image; same SLSA framework applies.
