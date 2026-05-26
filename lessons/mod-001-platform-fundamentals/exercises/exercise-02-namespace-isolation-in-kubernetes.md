# Exercise 02: Namespace Isolation in Kubernetes

## Objective

Design and implement a complete *namespace-per-tenant* isolation pattern for an ML platform on Kubernetes. By the end of this exercise, you will produce a set of YAML manifests that, applied to a cluster, provision a new tenant with: a namespace, ServiceAccount, RoleBindings, ResourceQuota, LimitRange, default-deny NetworkPolicy plus explicit-allow policies, and labels/annotations for cost allocation. You will also document the threat model your design defends against, and the threat model it deliberately does not.

This exercise is hands-on with YAML manifests but does not require you to apply them to a real cluster (though you can — `kind` and `minikube` work). The goal is *to internalize what a complete tenant-onboarding manifest looks like*, which is harder than it sounds.

## Learning Outcomes

By completing this exercise, you will:

- Write a complete Kubernetes namespace setup for a multi-tenant ML platform.
- Distinguish what namespaces *do* and *do not* isolate.
- Apply ResourceQuota, LimitRange, RBAC, and NetworkPolicy together as a system.
- Document the threat model your isolation defends against — and what it doesn't.
- Critique a colleague's incomplete tenant manifest.

## Prerequisites

- Read Lecture 03 (Multi-Tenancy Patterns) in full.
- Familiarity with Kubernetes basics: pod, service, namespace, deployment.
- Familiarity with YAML.
- Optional: a local Kubernetes cluster (`kind`, `minikube`, `k3d`) to actually apply the manifests.
- Optional: `kubectl` and `kustomize` on your path.

## Scenario

You are still at "Aurelia AI" (from Exercise 01). The platform team has decided that *every team gets its own namespace*. When a new team onboards, they get a tenant namespace with everything they need. You are writing the manifest template for that.

Today, you are provisioning a new tenant for the "ml-research" team:

- The team has 6 engineers, identified by an OIDC group `ml-research-engineers`.
- The team has one team lead, in OIDC group `ml-research-admins`.
- Their initial quota: 40 CPU cores, 200Gi memory, 4 GPUs, 50 PVCs, 100 pods.
- They can talk to the platform's services in the `platform-system` namespace.
- They can pull container images from a shared `internal-registry.aurelia.example.com` (running in `platform-system`).
- They can reach the company's internal Git server in the `git-system` namespace.
- They should NOT be able to reach any other tenant's namespace.
- They should NOT be able to make direct outbound calls to the public internet — outbound goes through an egress proxy in `platform-system`.

## Deliverables

By the end of this exercise, you will have created:

1. A `tenant-ml-research/` directory containing several YAML manifests (one per Kubernetes resource type).
2. A `kustomization.yaml` that assembles them.
3. A `threat-model.md` documenting what the isolation defends against.
4. A `verification.md` documenting how you would verify the isolation works (in cluster or via tools like `kubectl auth can-i`).
5. A short `critique.md` reviewing a *deliberately broken* colleague-supplied manifest at the end of this exercise.

---

## Part 1: Namespace and Labels (10 minutes)

### Task 1.1: Create the namespace manifest

Create `tenant-ml-research/00-namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-ml-research
  labels:
    # TODO: identify the tenant
    platform.aurelia.example.com/tenant: ml-research
    # TODO: cost allocation
    platform.aurelia.example.com/cost-center: cc-21044
    # TODO: tier (standard, premium, etc.) — choose one
    platform.aurelia.example.com/tier: standard
    # Pod Security Admission standards — choose levels
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
  annotations:
    platform.aurelia.example.com/owner-email: ml-research-leads@aurelia.example.com
    platform.aurelia.example.com/onboarded-at: "2026-05-22T00:00:00Z"
    platform.aurelia.example.com/provisioned-by: platform-cli
    platform.aurelia.example.com/documentation: |
      Tenant namespace for the ML Research team.
      See https://internal-docs/teams/ml-research for the team handbook.
```

**TODO**: Fill in the namespace manifest. Decide what Pod Security Admission level to enforce. (Hint: `restricted` is the safest but breaks many ML workloads that legitimately need things like `hostPath` for GPU drivers; `baseline` is the common compromise. Enforce `baseline`, audit/warn `restricted`, to see what's tripping the stricter level.)

### Task 1.2: Reflect on the labels

The labels you chose drive several downstream systems:

- Cost-allocation tools (Kubecost) read them to attribute spend.
- Observability tooling reads them to scope dashboards.
- Admission controllers can use them to enforce policy.

**TODO**: In `threat-model.md`, write a short paragraph on what would happen if the `platform.aurelia.example.com/tenant` label were *missing* or *wrong*. Could a malicious tenant set it? (Hint: who can edit namespace labels? Whose RBAC controls that?)

---

## Part 2: ServiceAccount and RBAC (20 minutes)

### Task 2.1: Workload ServiceAccount

Create `tenant-ml-research/10-serviceaccount.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: workload
  namespace: tenant-ml-research
  annotations:
    # Workload identity binding (example for EKS / IRSA).
    # The actual ARN depends on your cloud setup; we are not running
    # in a real account so this is illustrative.
    eks.amazonaws.com/role-arn: arn:aws:iam::000000000000:role/ml-research-workload
```

For non-AWS clouds, the annotation is different:

- **GKE / Workload Identity**: `iam.gke.io/gcp-service-account: ml-research@PROJECT.iam.gserviceaccount.com`
- **AKS / Workload Identity**: `azure.workload.identity/client-id: <GUID>`

**TODO**: Pick a cloud (or "cloud-neutral, no IAM") and write the annotation accordingly. Note in `threat-model.md`: what does this ServiceAccount allow when used by a pod? (It does *not* automatically grant Kubernetes RBAC — see Task 2.3 for that.)

### Task 2.2: Define platform-supplied Roles

In a real platform, common Roles are defined in `platform-system` as ClusterRoles, then RoleBindings bind them into each tenant namespace. We will skip the ClusterRole definitions (they live in the platform's own manifests) and reference standard names: `tenant-developer` and `tenant-admin`.

For this exercise, write the *RoleBinding* manifests, not the ClusterRole manifests.

Create `tenant-ml-research/20-rolebindings.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ml-research-engineers
  namespace: tenant-ml-research
subjects:
  - kind: Group
    name: ml-research-engineers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: tenant-developer
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ml-research-admins
  namespace: tenant-ml-research
subjects:
  - kind: Group
    name: ml-research-admins
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: tenant-admin
  apiGroup: rbac.authorization.k8s.io
```

**TODO**: Write these RoleBindings. Note that they bind a *ClusterRole* (a cluster-level definition) *into a single namespace* via RoleBinding — this is the "scope a generic ClusterRole into a tenant" pattern.

### Task 2.3: Pod-scoped permissions

The workload ServiceAccount you created earlier needs *its own* RBAC if pods in the namespace will call the Kubernetes API (e.g., a training operator that reads ConfigMaps).

For ML workloads that just run training code, the ServiceAccount typically does *not* need Kubernetes API access at all — the pod runs, talks to S3 (via workload identity), and exits. In that case, don't grant any Kubernetes RBAC to `workload`.

For workloads that *do* need API access, grant only the specific Roles needed. Example: a workload that reads platform ConfigMaps:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: workload-configmap-reader
  namespace: tenant-ml-research
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["platform-config", "tenant-config"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: workload-configmap-reader
  namespace: tenant-ml-research
subjects:
  - kind: ServiceAccount
    name: workload
    namespace: tenant-ml-research
roleRef:
  kind: Role
  name: workload-configmap-reader
  apiGroup: rbac.authorization.k8s.io
```

**TODO**: Decide whether `workload` needs Kubernetes API access at all. If yes, add the appropriate Role + RoleBinding. If no, document the choice in `threat-model.md`.

### Task 2.4: What the developer role should NOT include

A common mistake is to give developers permission to edit the namespace's quotas, network policies, or RoleBindings — which lets them silently escalate.

In `threat-model.md`, list the verbs/resources that `tenant-developer` should *not* be able to touch. At minimum:

- `resourcequotas` — managed by platform team
- `limitranges` — managed by platform team
- `networkpolicies` — managed by platform team
- `rolebindings` / `roles` — managed by platform team
- The namespace object itself — managed by platform team

**TODO**: Write a paragraph explaining why these are excluded and what would go wrong if they were not.

---

## Part 3: ResourceQuota and LimitRange (15 minutes)

### Task 3.1: ResourceQuota

Create `tenant-ml-research/30-resourcequota.yaml`:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-ml-research-quota
  namespace: tenant-ml-research
spec:
  hard:
    # CPU
    requests.cpu: "40"
    limits.cpu: "80"
    # Memory
    requests.memory: 200Gi
    limits.memory: 400Gi
    # GPUs
    requests.nvidia.com/gpu: "4"
    # Object counts
    pods: "100"
    services.loadbalancers: "0"        # tenants should never expose external LBs
    persistentvolumeclaims: "50"
    requests.storage: 5Ti
    # Ephemeral storage to prevent disk-fill attacks
    requests.ephemeral-storage: 500Gi
    limits.ephemeral-storage: 1Ti
```

**TODO**: Complete this manifest. Decide:

- Why is `services.loadbalancers: "0"`? (You don't want a tenant exposing internet-facing LBs; that's a platform-controlled action.)
- Should `requests.cpu` be different from `limits.cpu`? Why? (Burstable; users can specify higher limits than requests, but the team's total committed CPU is bounded.)
- How did you pick these numbers? (Document the assumption in `threat-model.md`.)

### Task 3.2: LimitRange

Create `tenant-ml-research/40-limitrange.yaml`:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: tenant-ml-research-defaults
  namespace: tenant-ml-research
spec:
  limits:
    - type: Container
      default:
        cpu: "1"
        memory: 2Gi
        ephemeral-storage: 1Gi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
        ephemeral-storage: 100Mi
      max:
        cpu: "16"
        memory: 64Gi
        ephemeral-storage: 200Gi
      min:
        cpu: 10m
        memory: 16Mi
        ephemeral-storage: 10Mi
    - type: PersistentVolumeClaim
      max:
        storage: 1Ti
      min:
        storage: 1Gi
```

**TODO**: Complete this manifest. Reflect: a user who submits a pod without specifying resources will get the `default` and `defaultRequest` values. Is this what you want? (Sometimes yes — guarantee everyone gets *something*; sometimes no — better to force users to be explicit.)

In `threat-model.md`, document: if a tenant submits 100 pods each with `defaultRequest`, what's the total committed CPU/memory? Does that match the quota? (100 × 100m = 10 CPU; well within quota. 100 × 128Mi ≈ 12.5Gi; well within quota.)

### Task 3.3: GPU-specific concerns

GPUs are special. The LimitRange above doesn't say anything about GPUs because LimitRange `max` doesn't apply to extended resources by default. Check the Kubernetes docs.

**TODO**: Document in `threat-model.md`: how do you prevent a single pod from requesting all 4 GPUs at once? (You can rely on `requests.nvidia.com/gpu: "4"` in the quota — but that lets *one* pod consume the whole quota. If you want to prevent that, you need an admission controller. Note this as a deliberate gap.)

---

## Part 4: NetworkPolicy (25 minutes)

This is the most error-prone part. Take your time.

### Task 4.1: Default-deny

Create `tenant-ml-research/50-networkpolicy-default-deny.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: tenant-ml-research
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

This denies all ingress and egress for all pods in the namespace. Now you have to *opt back in* to the connections that should be allowed.

### Task 4.2: Allow DNS

DNS is the first thing you need:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: tenant-ml-research
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

**TODO**: Create this manifest. Note that the selector path is a bit subtle — it's "pods in the `kube-system` namespace with label `k8s-app: kube-dns`."

### Task 4.3: Allow platform services

The tenant needs to reach the platform's APIs (registry, training submission, etc.) which live in `platform-system`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-platform-system
  namespace: tenant-ml-research
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: platform-system
      ports:
        - protocol: TCP
          port: 443      # HTTPS APIs
        - protocol: TCP
          port: 80       # HTTP (e.g., internal registry — but prefer 443)
```

**TODO**: Create this manifest. Decide: is "any pod in platform-system on 443" too broad? (Probably for a hard-multi-tenancy posture. Tighten by adding `podSelector` to allow only specific platform services.)

### Task 4.4: Allow git server

The tenant needs to pull code from the internal Git server:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-git-system
  namespace: tenant-ml-research
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: git-system
      ports:
        - protocol: TCP
          port: 22       # SSH for git
        - protocol: TCP
          port: 443      # HTTPS for git
```

**TODO**: Create this manifest.

### Task 4.5: Block egress to other tenants and internet

The default-deny from Task 4.1 already blocks these. The question is: did you *also* allow them through any of the explicit-allow policies?

NetworkPolicies are **additive** — if any policy allows a connection, the connection is allowed. So if `allow-egress-platform-system` happens to allow pods in another tenant's namespace because they share a label, you've leaked.

**TODO**: Review your allow policies. For each one, write in `threat-model.md`: "this policy permits *only* the following connections: …" and confirm no other tenant is included by accident.

### Task 4.6: Allow ingress from platform services

Sometimes platform services need to *call* tenant pods (e.g., a webhook receiver, a health checker). Selectively allow:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-from-platform-system
  namespace: tenant-ml-research
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: platform-system
```

**TODO**: Decide whether this is needed for your scenario. If the platform never initiates calls into the tenant's pods, leave ingress fully blocked.

### Task 4.7: Egress proxy for outbound

The scenario says outbound goes through an egress proxy in `platform-system`. That means tenant pods should *not* reach the public internet directly.

Your NetworkPolicy already blocks this (default-deny + no allow). But to make it usable, tenant code must be configured with an `HTTPS_PROXY` environment variable pointing at the proxy. That's a *user-education* problem, not a NetworkPolicy problem.

**TODO**: In `threat-model.md`, document: "Outbound to the internet from this namespace requires use of the egress proxy at `proxy.platform-system.svc.cluster.local:8080`. Direct outbound is blocked at the NetworkPolicy layer. Users must set `HTTPS_PROXY` for libraries that do not auto-detect."

---

## Part 5: Default ConfigMap for tenant settings (5 minutes)

Many platforms supply a default ConfigMap that tenant workloads can mount to discover platform endpoints, environment, etc.

Create `tenant-ml-research/60-configmap-platform.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: platform-config
  namespace: tenant-ml-research
data:
  platform.api.endpoint: "https://api.platform-system.svc.cluster.local:443"
  registry.endpoint: "https://registry.platform-system.svc.cluster.local:443"
  proxy.https: "http://proxy.platform-system.svc.cluster.local:8080"
  proxy.http: "http://proxy.platform-system.svc.cluster.local:8080"
  tenant.name: "ml-research"
  tenant.cost-center: "cc-21044"
  tenant.tier: "standard"
```

**TODO**: Create this ConfigMap. Note: even though the developer Role grants read access to ConfigMaps, this one is in their namespace and they can read it. They cannot edit it (the developer Role we conceptualized only allows reading platform-supplied configmaps).

---

## Part 6: Kustomize Assembly (5 minutes)

Create `tenant-ml-research/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - 00-namespace.yaml
  - 10-serviceaccount.yaml
  - 20-rolebindings.yaml
  - 30-resourcequota.yaml
  - 40-limitrange.yaml
  - 50-networkpolicy-default-deny.yaml
  - 51-networkpolicy-allow-dns.yaml
  - 52-networkpolicy-allow-platform-system.yaml
  - 53-networkpolicy-allow-git-system.yaml
  - 60-configmap-platform.yaml

commonLabels:
  platform.aurelia.example.com/managed-by: platform-cli
  platform.aurelia.example.com/tenant: ml-research

commonAnnotations:
  platform.aurelia.example.com/version: "v1"
```

**TODO**: Adjust filenames to match what you actually created. Run `kubectl kustomize tenant-ml-research/` (if you have kustomize / kubectl) to validate the assembly. If you don't have the tools, mentally trace through and check that filenames match.

---

## Part 7: Threat Model (15 minutes)

Now consolidate everything you've written into `threat-model.md`.

### Task 7.1: What does the isolation defend against?

List threats your design *does* defend against:

1. **Accidental name collisions.** Two teams both have a `model-trainer` deployment — namespaces prevent collision.
2. **Accidental resource exhaustion.** ResourceQuota prevents one team from consuming all CPU/GPU.
3. **Accidental cross-tenant network access.** NetworkPolicies block tenant-to-tenant communication.
4. **Misuse of RBAC by developers.** Developer Role excludes destructive verbs; only admins can modify some resources.
5. **Unauthenticated requests.** RoleBindings only grant access to authenticated members of specific OIDC groups.
6. **Direct internet egress.** Default-deny egress + proxy-only outbound makes data exfiltration require deliberate evasion.

### Task 7.2: What does the isolation NOT defend against?

This is more important. Be honest:

1. **Kernel escapes.** A pod that exploits a kernel bug to escape its container can affect other pods on the same node. Namespaces are *not* a security boundary at the kernel level. *Mitigation*: dedicated node pools or runtime sandboxes (gVisor, Kata) for hostile tenants; we have not added these.
2. **API server abuse.** A tenant with many controllers can degrade the API server for everyone. *Mitigation*: API priority/fairness; not configured here.
3. **Image supply chain.** Tenants pull images from the internal registry, but the internal registry's content is not verified. A malicious image could exfiltrate data within its allowed network paths. *Mitigation*: image signing, admission control on image sources; not in this exercise.
4. **Secret leakage in pod specs.** The platform's developer Role allows reading Secrets in the namespace. A user who logs a Secret value can leak it. *Mitigation*: external secret management; out of scope here.
5. **Side channels.** Two tenants on the same node may infer information about each other through CPU caches, memory bandwidth, etc. *Mitigation*: dedicated node pools for sensitive tenants; not configured.
6. **Cluster-admin compromise.** If a cluster admin is compromised, the entire cluster is compromised. *Mitigation*: minimize cluster-admin grants, monitor admin actions, MFA — operational concerns, not in YAML.
7. **Quota exhaustion of cluster-wide resources.** Some resources (LoadBalancer IPs, certain CRDs) are not namespaced. *Mitigation*: case-by-case admission controllers.

**TODO**: Add this section to `threat-model.md`. Be exhaustive — listing what you *don't* defend against is more useful than listing what you do.

### Task 7.3: Threat tier classification

Per Lecture 03, classify your tenancy as soft, hard, or hostile. Defend the classification.

For the Aurelia scenario (internal company, ~30 engineers, all on payroll), **soft multi-tenancy** is the right classification. Your YAML is appropriate for soft; if Aurelia ever onboards external customers as tenants, you would need to add hard- or hostile-tenancy controls.

**TODO**: Write one paragraph on this.

---

## Part 8: Verification (10 minutes)

Document how you'd verify your isolation works. This goes in `verification.md`.

### Task 8.1: kubectl auth can-i

For each role, write the `kubectl auth can-i` commands that should succeed and the ones that should fail.

```bash
# As an ml-research-engineer, I should be able to create pods in my namespace:
kubectl --as=alice@aurelia --as-group=ml-research-engineers \
  auth can-i create pods -n tenant-ml-research
# Expected: yes

# As the same user, I should NOT be able to edit the quota:
kubectl --as=alice@aurelia --as-group=ml-research-engineers \
  auth can-i patch resourcequotas -n tenant-ml-research
# Expected: no

# As the same user, I should NOT be able to create pods in another tenant:
kubectl --as=alice@aurelia --as-group=ml-research-engineers \
  auth can-i create pods -n tenant-other-team
# Expected: no

# As an ml-research-admin, I CAN edit my namespace's pods:
kubectl --as=bob@aurelia --as-group=ml-research-admins \
  auth can-i delete deployments -n tenant-ml-research
# Expected: yes

# As an ml-research-admin, I should NOT be able to edit the quota:
kubectl --as=bob@aurelia --as-group=ml-research-admins \
  auth can-i patch resourcequotas -n tenant-ml-research
# Expected: no (only platform-admin can)
```

**TODO**: Write at least 10 `kubectl auth can-i` commands covering positive and negative cases.

### Task 8.2: NetworkPolicy verification

NetworkPolicy is harder to verify because it depends on the CNI. The standard approach: deploy two test pods, run `nc` or `curl` between them, observe what works.

Write a script (or pseudocode) that:

```bash
# Deploy a test pod in tenant-ml-research
kubectl run testpod --image=alpine --namespace=tenant-ml-research -- sleep 3600

# Verify DNS works
kubectl exec -n tenant-ml-research testpod -- nslookup kubernetes.default.svc.cluster.local
# Expected: success

# Verify platform-system reachable
kubectl exec -n tenant-ml-research testpod -- \
  wget -qO- http://api.platform-system.svc.cluster.local
# Expected: connection succeeds (may return non-200 but TCP works)

# Verify other tenant NOT reachable
kubectl exec -n tenant-ml-research testpod -- \
  wget -qO- --timeout=5 http://service.tenant-other.svc.cluster.local
# Expected: timeout / connection refused (NetworkPolicy blocking)

# Verify external internet NOT reachable
kubectl exec -n tenant-ml-research testpod -- \
  wget -qO- --timeout=5 https://example.com
# Expected: timeout (no egress allowed)

# Cleanup
kubectl delete pod testpod -n tenant-ml-research
```

**TODO**: Write at least 5 such verification commands.

### Task 8.3: Quota verification

```bash
# Try to deploy a pod that requests more than quota allows
cat <<EOF | kubectl apply -n tenant-ml-research -f -
apiVersion: v1
kind: Pod
metadata:
  name: huge-pod
spec:
  containers:
    - name: c
      image: alpine
      command: ["sleep", "3600"]
      resources:
        requests:
          cpu: "100"  # exceeds quota of 40
          memory: 4Gi
EOF
# Expected: error from server: "exceeded quota: tenant-ml-research-quota"

# Try to deploy a pod with no resources specified
# (LimitRange should fill in defaults)
cat <<EOF | kubectl apply -n tenant-ml-research -f -
apiVersion: v1
kind: Pod
metadata:
  name: small-pod
spec:
  containers:
    - name: c
      image: alpine
      command: ["sleep", "3600"]
EOF
# Expected: succeeds; describe the pod to see defaults applied
kubectl describe pod small-pod -n tenant-ml-research | grep -A2 Limits:
```

**TODO**: Write at least 3 verification commands for quota / LimitRange.

---

## Part 9: Critique a Colleague's Manifest (15 minutes)

Below is a deliberately incomplete manifest that a junior engineer submitted as their attempt at namespace isolation. Critique it in `critique.md`.

```yaml
# === Submitted by junior engineer for review ===
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-foo
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: foo-team-binding
  namespace: tenant-foo
subjects:
  - kind: Group
    name: foo-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: foo-quota
  namespace: tenant-foo
spec:
  hard:
    requests.cpu: "1000"
    requests.memory: 1Ti
    pods: "10000"
```

**TODO**: In `critique.md`, write a numbered list of issues with this manifest. Aim for at least 6 issues. For each, explain the problem and propose a fix.

Hints (don't read until you've tried):

<details>
<summary>Click for hints</summary>

1. The RoleBinding references `cluster-admin`. Even though it's a *RoleBinding* (scoped to namespace), bind-cluster-admin-into-namespace grants ClusterRole privileges *within the namespace*, which is wildly too broad. The user can do anything in their namespace, including editing the quota. Use a narrower `tenant-developer` or `tenant-admin`.
2. No `LimitRange` — pods without resource specs will not get sensible defaults and may either consume too much or be rejected for being too small.
3. No `NetworkPolicy` — full pod-to-pod connectivity, including to other tenants.
4. The quota's `pods: "10000"` is absurd; combined with `requests.cpu: "1000"`, this lets one tenant consume vastly more than reasonable.
5. No `ServiceAccount` for workloads — pods will use `default`, which has no IAM binding, so workloads can't access cloud resources.
6. No labels on the namespace — cost allocation and observability won't work.
7. No Pod Security Admission level set — pods can run as root with hostPath mounts.
8. Quota only covers `requests.cpu`, `requests.memory`, `pods` — no limit on GPUs, persistent volumes, services, etc.

</details>

---

## Common Pitfalls

1. **Forgetting DNS.** A default-deny NetworkPolicy that doesn't allow DNS breaks every pod. Add the DNS allow first.
2. **Over-broad RoleBindings.** Binding `cluster-admin` (or `admin`, or `edit`) into a namespace is sloppy. Define narrower roles.
3. **Quotas without LimitRanges.** ResourceQuota requires every pod to specify resources, or LimitRange defaults to kick in. Without LimitRange, pods without explicit resources are rejected — confusing for users.
4. **Labels missing.** Without consistent labels, cost allocation and observability don't work. Label everything.
5. **Network policies that accidentally allow too much.** A common bug: `namespaceSelector: {}` matches *all* namespaces, not none. Be precise with selectors.
6. **Forgetting Pod Security Admission.** Without PSA labels on the namespace, pods can run privileged. Always set `enforce`/`audit`/`warn` levels.

---

## Reflection Questions

In `threat-model.md` under a "Reflection" heading, answer:

1. A new requirement comes in: the ml-research team needs to call an external API (GitHub Webhooks). How does your design accommodate this? What's the change scope?
2. A new requirement: a *single* user from another team needs read-only access to ml-research's namespace for an audit. How do you grant that? What RBAC change is needed?
3. The platform team wants to extract per-tenant cost. What information are they reading off your manifests? What would break the cost report?
4. Your tenant manifest is one team. The platform has 30+ teams. How do you avoid copy-paste drift across 30 nearly-identical manifests? (Hint: think about templating, GitOps, helm charts, or kustomize bases.)
5. Six months from now, a security review notes that "namespaces are not a kernel boundary." How does your platform's security posture change in response? (Possible answers: add gVisor, add dedicated node pools, accept the risk because tenants are trusted.)

---

## Self-Assessment

- [ ] Can I list every kind of Kubernetes resource I created and explain why it's needed?
- [ ] Can I explain why default-deny NetworkPolicy is the right starting point?
- [ ] Can I name three things namespaces do *not* isolate?
- [ ] Can I write a single `kubectl auth can-i` command from memory?
- [ ] Have I identified at least 5 issues in the colleague's manifest?

If yes to all, you're done.

---

## Suggested Time Allocation

| Section | Time |
| --- | --- |
| Part 1: Namespace and Labels | 10 min |
| Part 2: ServiceAccount and RBAC | 20 min |
| Part 3: ResourceQuota and LimitRange | 15 min |
| Part 4: NetworkPolicy | 25 min |
| Part 5: ConfigMap | 5 min |
| Part 6: Kustomize Assembly | 5 min |
| Part 7: Threat Model | 15 min |
| Part 8: Verification | 10 min |
| Part 9: Critique | 15 min |
| **Total** | **120 min** |

If you have a local cluster, applying your manifests and running the verification commands adds another 30 minutes but is worth it — there's a difference between "I think this works" and "I watched it work."

---

## Where to Go from Here

You have now written a complete tenant-onboarding manifest. In production this would live in a `tenants/` directory in a GitOps repo, with one folder per tenant. When a new team is onboarded, the platform CLI generates a new folder with their settings, opens a PR, and a platform admin reviews and merges.

Module 03 (Kubernetes for ML) will dig into the runtime aspects of this design: how the scheduler interacts with quotas, how priority and preemption work, how to run training jobs in tenant namespaces using a job controller, and how to handle GPU-specific concerns. Module 09 (Security) will revisit RBAC and NetworkPolicy at depth.

For now, this exercise has taught you to *think in manifests* about multi-tenancy. Move on to Exercise 03.

---

## Appendix: Quick-reference cheatsheet

A one-page reference for "what a minimal sane tenant namespace looks like." Save this somewhere; you'll come back to it.

```yaml
# Minimum viable tenant namespace, in dependency order:
#  1. Namespace (with tenant + cost-center labels + PSA enforce)
#  2. ServiceAccount (workload identity bound externally)
#  3. RoleBindings (developer + admin groups bound to namespace-scoped roles)
#  4. ResourceQuota (capping CPU / memory / GPU / pods / storage)
#  5. LimitRange (per-container defaults / max / min)
#  6. NetworkPolicy: default-deny ingress + egress
#  7. NetworkPolicy: allow DNS
#  8. NetworkPolicy: allow egress to specific platform namespaces
#  9. NetworkPolicy: (optional) allow ingress from platform namespaces
# 10. ConfigMap: platform-supplied endpoints / proxy settings

# Number labels (1-10) match step order for "apply in this order."
```

The reason the order matters: namespaces before resources in them, RBAC before workloads (so the workload identity is established), quotas/limits before network (so the resources are sized), default-deny before allow lists (so misconfiguration is fail-closed, not fail-open).

A common rollout mistake is to apply manifests in alphabetical filename order without checking dependencies. If your `kustomization.yaml` lists them in dependency order (with numeric prefixes), kustomize handles the rest. We strongly recommend the numeric-prefix filename pattern shown in this exercise.
