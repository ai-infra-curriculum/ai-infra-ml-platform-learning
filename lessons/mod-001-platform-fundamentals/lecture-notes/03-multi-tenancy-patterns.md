# Lecture 03: Multi-Tenancy Patterns

## Table of Contents

1. [Introduction](#introduction)
2. [What Multi-Tenancy Means](#what-multi-tenancy-means)
3. [The Isolation Dimensions](#the-isolation-dimensions)
4. [Kubernetes Namespace Isolation](#kubernetes-namespace-isolation)
5. [Resource Quotas and LimitRanges](#resource-quotas-and-limitranges)
6. [Network Policy and Service Boundaries](#network-policy-and-service-boundaries)
7. [Identity, Authentication, and RBAC](#identity-authentication-and-rbac)
8. [Cost Allocation and Chargeback](#cost-allocation-and-chargeback)
9. [Data Isolation](#data-isolation)
10. [Noisy Neighbors and Fair Sharing](#noisy-neighbors-and-fair-sharing)
11. [Common Failure Modes](#common-failure-modes)
12. [Summary](#summary)

---

## Introduction

A platform with one tenant is a personal workstation. A platform with many tenants is a *platform*. Multi-tenancy is the defining operational property that distinguishes a platform from bespoke tooling, and it is also where most of the platform engineer's hardest problems live.

When two tenants share a platform, several questions immediately arise:

- Can tenant A see tenant B's data? (Authorization.)
- Can tenant A use up all the resources, leaving tenant B with none? (Quotas and fairness.)
- Can tenant A's misbehaving workload crash the platform for tenant B? (Isolation and blast radius.)
- Whose budget pays for what? (Cost allocation.)
- When tenant A makes a request, does the platform know it was tenant A? (Identity.)

If any of these questions has the wrong answer for your context, you have a multi-tenancy bug — possibly a security bug. ML platforms in particular get burned here because training workloads often run with broad access (to read large datasets) and consume large resources (GPUs), making the blast radius of mistakes large.

This chapter introduces the concepts. We do not implement multi-tenancy from scratch in this module — that work is spread across Modules 03 (orchestration), 09 (security), and 10 (governance). The goal of *this* chapter is to give you the vocabulary, the patterns, and the failure modes, so when you read a multi-tenancy design doc you can engage with it critically.

By the end you should be able to:

- Distinguish the seven dimensions of isolation that matter for ML platforms.
- Sketch a Kubernetes-namespace-per-tenant architecture, including its limits.
- Read a Kubernetes `ResourceQuota` / `LimitRange` manifest and explain what it enforces.
- Identify common multi-tenancy failure modes (noisy neighbors, cost leakage, escalation paths) and name at least one mitigation for each.
- Articulate when to use *strong* vs *weak* multi-tenancy and the tradeoffs.

---

## What Multi-Tenancy Means

### A working definition

A platform is **multi-tenant** if multiple distinct *tenants* — typically teams, but possibly individual users, customers, or organizational units — share underlying resources while being isolated from each other along dimensions that matter.

"Dimensions that matter" is doing a lot of work. The interesting design question is *which dimensions matter for this platform*, given its threat model, its blast radius, and its users' expectations.

### Soft, hard, and hostile tenancy

It is useful to distinguish three flavors of multi-tenancy:

- **Soft (cooperative) multi-tenancy.** All tenants are inside the same company / trust boundary. They may not coordinate, but they are not actively adversarial. Most internal ML platforms are in this category. The goal is to prevent *accidents*, not attacks.
- **Hard multi-tenancy.** Tenants are within a broad trust boundary (e.g., same company) but you must defend against *aggressive accidents* — a runaway training job that consumes 100% of available GPUs, credentials accidentally checked into a notebook, etc. Most mature ML platforms aspire to this level.
- **Hostile multi-tenancy.** Tenants are explicitly untrusted (e.g., customers in a public SaaS, students in a university lab, contractors in a regulated environment). You must defend against deliberate attacks. This is a much harder problem than hard multi-tenancy.

The threat model shapes what isolation is necessary. Don't over-engineer for hostile tenancy if you are running an internal ML platform; the cost (operational complexity, performance overhead, user friction) is high and may not be justified.

That said, *don't under-engineer either*: the cost of a security breach can be vastly higher than the cost of slightly stronger isolation.

A useful test: ask "what is the worst-case outcome if tenant A maliciously tried to access tenant B's data or starve them of compute?" If the answer is "the company is on the front page of the news," you need at least hard multi-tenancy. If the answer is "we'd have an awkward Slack thread," soft tenancy may be acceptable.

### Tenancy granularity

Who is a tenant?

- A whole **business unit** (sales, marketing, finance)?
- A **team** (the churn-modeling team)?
- A **user** (one data scientist)?
- A **workload** (one training job)?

Different platforms make different choices. ML platforms commonly use **team** as the tenant unit, because:

- ML work is collaborative; data scientists in a team usually share datasets and models.
- Cost allocation is usually done at the team / cost-center level.
- Access control to sensitive data is usually team-shaped.

But it's not the only choice. Some platforms make *individual users* the tenant for development workloads (each user has their own namespace) and *teams* the tenant for production workloads. This hybrid is common.

The choice has implications:

- Per-user tenancy gives the strongest isolation but multiplies the operational cost (many more namespaces, many more quotas).
- Per-team tenancy is the common middle ground.
- Per-business-unit tenancy is coarse; it doesn't prevent intra-BU conflicts.

You usually do not make this choice once and forever. It evolves.

---

## The Isolation Dimensions

ML platforms must consider isolation along (at least) seven dimensions. We call this the "seven faces of multi-tenancy." Each requires its own mechanism; getting any of them wrong is bad in its own way.

| Dimension | What is isolated | Typical mechanism |
| --- | --- | --- |
| **Compute** | CPU, memory, GPU | Kubernetes ResourceQuota, PriorityClass, scheduler config |
| **Storage** | Persistent volumes, dataset access | Storage classes, S3 IAM policies, bucket policies |
| **Network** | Service-to-service traffic | NetworkPolicy, service mesh, namespace-scoped DNS |
| **Identity** | Who is calling | OIDC, service accounts, IAM roles |
| **Authorization** | What they can do | RBAC, OPA / Gatekeeper, fine-grained policy |
| **Observability** | Logs, metrics, traces | Per-tenant labels, scoped dashboards |
| **Cost** | Cloud spend attribution | Resource labels, billing tags, chargeback model |

Three observations:

1. **You need every dimension.** Skipping any one is a silent multi-tenancy bug. You will eventually discover it the hard way.
2. **The dimensions interact.** A tenant who can see another tenant's logs (observability leak) can often deduce data they shouldn't see. A tenant with broad network access (network leak) can often access services across tenant boundaries. Treat the dimensions as a system.
3. **Different organizations weight them differently.** A regulated industry (healthcare, finance) typically over-invests in observability and authorization. A startup typically under-invests in cost allocation until the bill arrives. Calibrate to your context.

The rest of this chapter walks each dimension in more depth, with the heaviest weight on compute, network, and identity — the ones ML platform engineers spend the most time on.

---

## Kubernetes Namespace Isolation

Most modern ML platforms run on Kubernetes, and most use **namespaces** as the primary tenancy boundary. A namespace is a Kubernetes-native abstraction that scopes names of resources (pods, services, secrets, configmaps, deployments) so that tenant A's `my-trainer` does not collide with tenant B's `my-trainer`.

Namespaces alone provide:

- **Name scoping** (no collision within a namespace).
- **A label/selector boundary** for RBAC and policy.
- **A unit for cleanup** (delete a namespace, you delete the workloads in it).

Namespaces alone *do not* provide, out of the box:

- **Resource isolation.** A pod in namespace A and a pod in namespace B can land on the same node and compete for CPU/memory unless you add scheduling rules.
- **Network isolation.** Pods in different namespaces can reach each other by default unless you add NetworkPolicies.
- **Identity isolation.** Service accounts are namespaced, but RBAC roles are not automatically scoped.
- **Strong security boundaries.** A compromised pod can sometimes escape to the node and then affect other namespaces; namespaces are not a security boundary in the kernel sense.

So a "namespace-per-tenant" architecture is the *starting point*, not the *finish line*.

### A typical namespace-per-tenant layout

```
cluster
├── namespace: platform-system           # platform team's own services
├── namespace: tenant-team-alpha         # team alpha's workloads
├── namespace: tenant-team-beta          # team beta's workloads
├── namespace: tenant-team-gamma         # team gamma's workloads
└── namespace: tenant-team-delta         # team delta's workloads
```

Each tenant namespace contains:

- The tenant's training jobs, batch jobs, and inference services.
- A `ServiceAccount` that pods in the namespace use to talk to the platform's services.
- A `ResourceQuota` capping how much CPU/memory/GPU the tenant can use.
- One or more `RoleBinding`s granting tenant members `Role`s scoped to the namespace.
- `NetworkPolicy` resources controlling traffic into and out of the namespace.

A `platform-system` namespace hosts the platform team's own services (control plane, registry, dashboards, etc.) and has different rules — it generally needs to be reachable from all tenants, while tenants cannot reach each other.

### Worked example: minimal tenant namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-team-alpha
  labels:
    platform.example.com/tenant: team-alpha
    platform.example.com/tier: standard
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: workload
  namespace: tenant-team-alpha
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-alpha-members-edit
  namespace: tenant-team-alpha
subjects:
  - kind: Group
    name: team-alpha-engineers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
```

This grants members of the OIDC group `team-alpha-engineers` the built-in `edit` ClusterRole *scoped to the `tenant-team-alpha` namespace*. They can create pods, jobs, configmaps, etc., but only in their own namespace.

(Note: the `edit` ClusterRole is intentionally broad. Many platforms write their own narrower Role; this example is illustrative.)

### Limits of the namespace-per-tenant pattern

The pattern works well up to a few hundred tenants and breaks down at higher scales. Symptoms:

- **API server pressure.** Many namespaces with many resources can stress the API server, especially with controllers that watch all namespaces.
- **RBAC sprawl.** Each tenant's `RoleBinding`s, `ServiceAccount`s, `NetworkPolicy`s must be created and kept in sync. Without automation, this rapidly becomes unmanageable.
- **Operational visibility.** "Show me everything happening in the cluster" becomes hard when "everything" is spread across hundreds of namespaces.

At scale, platform teams adopt one of three responses:

1. **Multi-cluster.** Each large tenant or tenant group gets its own cluster. Hard isolation but high operational cost.
2. **Virtual clusters** (e.g., vCluster). A "cluster" inside a namespace; tenants get a strongly-scoped API server.
3. **Hierarchical namespaces** (HNC, the Kubernetes Hierarchical Namespace Controller). A tree of namespaces with policy inheritance.

These are addressed in Module 03 (Kubernetes for ML), so we'll only mention them here. For Module 01, internalize the basic shape.

### A short policy aside: namespaces are not a security boundary

The Kubernetes documentation is explicit about this. A namespace is a *naming* boundary, not a *security* boundary. A pod in namespace A that escapes its container (via a kernel exploit, a misconfigured privileged pod, an excessive `hostPath` mount) can affect pods in other namespaces on the same node.

For strong security isolation between tenants, you need additional controls:

- Separate node pools per tenant (so tenants don't share nodes).
- Pod Security Admission (or its older sibling PodSecurityPolicy, which is deprecated).
- A runtime sandbox (gVisor, Kata, Firecracker) for further isolation.
- Image admission control (only trusted images can run).

These are decisions you make based on your threat model. Soft-tenancy clusters generally don't bother with all of this; hard-tenancy clusters do most of it; hostile-tenancy clusters do all of it.

---

## Resource Quotas and LimitRanges

Two Kubernetes primitives, two different jobs.

### ResourceQuota

A `ResourceQuota` caps the *total* resources a namespace can consume. Examples:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-alpha-quota
  namespace: tenant-team-alpha
spec:
  hard:
    requests.cpu: "200"
    requests.memory: "500Gi"
    requests.nvidia.com/gpu: "8"
    limits.cpu: "400"
    limits.memory: "1000Gi"
    persistentvolumeclaims: "20"
    pods: "100"
    services.loadbalancers: "2"
```

What this means:

- The sum of all pods' `requests.cpu` in the namespace cannot exceed 200 cores.
- The sum of all pods' `requests.memory` cannot exceed 500GiB.
- The sum of GPUs requested cannot exceed 8.
- Limits are also capped (200 CPU requested → 400 CPU limit max).
- No more than 20 PVCs, 100 pods, 2 LoadBalancer services.

When a tenant tries to submit a pod that would push them over quota, the Kubernetes API server rejects the request with a clear error.

ResourceQuota is the primary lever for **compute fairness**. Without it, a tenant with a bad training script can request 10,000 GPUs and exhaust the cluster.

### LimitRange

A `LimitRange` sets default and max *per-pod* resource limits. Example:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: team-alpha-pod-limits
  namespace: tenant-team-alpha
spec:
  limits:
    - type: Container
      default:
        cpu: "1"
        memory: "2Gi"
      defaultRequest:
        cpu: "100m"
        memory: "256Mi"
      max:
        cpu: "16"
        memory: "64Gi"
      min:
        cpu: "10m"
        memory: "16Mi"
```

What this means:

- If a container is submitted without a CPU limit, it gets 1 core.
- If submitted without a memory request, it gets 256MiB.
- No single container can request more than 16 cores or 64GiB memory.
- No container can request less than 10m CPU or 16MiB memory.

LimitRange protects the cluster from pathological pod specifications. A user who writes "request 0.001 CPU and 1MiB memory" (and then runs a real workload that consumes much more) will trip the min. A user who tries "request 256 cores and 1TiB memory" will trip the max.

### How they compose

`LimitRange` says "no single container can use more than X." `ResourceQuota` says "the namespace as a whole cannot use more than Y." Together they bound both *per-workload* and *per-tenant* consumption.

A common starter configuration:

- `LimitRange` defaults: tiny, so users have to explicitly ask for more (signaling intent).
- `LimitRange` max: large enough for the largest reasonable workload (e.g., 32 CPU, 256GiB memory, 8 GPUs per pod).
- `ResourceQuota`: tuned per-tenant based on their budget and historical usage.

### GPUs specifically

GPUs are the resource ML platforms care most about. Some specifics:

- GPUs are typically modeled as `nvidia.com/gpu` (or equivalent for AMD, AWS Inferentia, TPUs).
- GPUs are *whole-unit*: you can't request 0.3 GPUs (without MIG or time-slicing tricks). A pod requests 1, 2, 4, or 8 whole GPUs.
- GPUs are *scarce and expensive*. Quotas matter much more for GPUs than for CPUs.
- The cluster scheduler must know about GPU node taints so non-GPU pods don't take GPU nodes.

A typical ML-platform-flavored quota:

```yaml
spec:
  hard:
    requests.cpu: "200"
    requests.memory: "500Gi"
    requests.nvidia.com/gpu: "16"
    requests.ephemeral-storage: "2Ti"
```

Plus, on the cluster level, GPU nodes are tainted:

```yaml
spec:
  taints:
    - key: nvidia.com/gpu
      value: "true"
      effect: NoSchedule
```

And training pods tolerate:

```yaml
tolerations:
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
```

This way, only pods explicitly designed for GPU work end up on GPU nodes. We discuss GPU scheduling in detail in Module 03.

---

## Network Policy and Service Boundaries

By default in Kubernetes, every pod can talk to every other pod, in any namespace. That is *not* multi-tenant safe.

### NetworkPolicy

A `NetworkPolicy` is a Kubernetes resource that, when supported by your CNI plugin (Calico, Cilium, etc.), restricts pod-to-pod traffic. A "default deny" policy is a common starting point:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: tenant-team-alpha
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

This denies all ingress and egress for all pods in the namespace.

Then add specific *allows*:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-platform-services
  namespace: tenant-team-alpha
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
          port: 443
```

This allows pods in `tenant-team-alpha` to reach pods in `platform-system` on TCP/443 (for talking to the platform's HTTPS APIs) and nothing else.

You also typically allow DNS (so name resolution works) and any specific cross-tenant communication you need.

The pattern is: **start with default-deny, then add explicit allows for known good paths.** This is the inverse of the default behavior, and it's the safer one.

### Service mesh

A service mesh (Istio, Linkerd, Consul Connect) adds an *additional* layer of identity-based authorization on top of network policy. With a mesh:

- Every service has a cryptographic identity (typically via SPIFFE / mTLS).
- Authorization policies can be written as "service A in tenant alpha can call service B in tenant beta" — at the application identity layer, not just the IP layer.

For ML platforms, a service mesh is often overkill at small scale and a real value-add at large scale (especially when you need cross-cluster identity).

### Egress policy

A topic that ML platforms in particular care about: **outbound traffic from training jobs**.

Training jobs may need to pull data, models, or dependencies from external services (GitHub, PyPI, HuggingFace Hub, etc.). They also may *not* need to — and if they don't, allowing arbitrary egress is a data-exfiltration risk.

Common patterns:

- **Default deny egress** at the namespace level, with allow lists for specific known destinations.
- **An egress proxy** that all outbound traffic must go through, with auth and logging.
- **Cached internal mirrors** (a PyPI proxy, an artifact registry) so training jobs don't need direct external egress at all.

The right pattern depends on your security posture. Some industries (finance, healthcare) require all outbound traffic to be logged.

---

## Identity, Authentication, and RBAC

### Who is calling?

Every action on the platform should be traceable to an *identity*. There are usually two flavors:

- **Human identities**: data scientists and engineers who interact with the platform interactively. Typically authenticated via OIDC (with the company SSO as the IdP).
- **Workload identities**: pods, jobs, services that run on behalf of a tenant. Typically a ServiceAccount in Kubernetes, possibly federated to a cloud IAM role via Workload Identity / IRSA.

A common design:

- Humans log into the platform via OIDC → get a token.
- That token's claims include the human's group memberships (e.g., `team-alpha-engineers`).
- The platform uses those claims to determine what the human can do (which tenants they belong to).
- When a human submits a workload, the workload runs as a ServiceAccount in their tenant's namespace.
- The ServiceAccount has IAM permissions scoped to that tenant's resources (S3 buckets, BigQuery datasets, etc.).

This way, *the human's permissions* and *the workload's permissions* are separate but linked. A workload can only access things the human's tenant can access, but the workload doesn't carry the human's credentials.

### RBAC in Kubernetes

Kubernetes RBAC has four resource types:

- **Role**: a set of permissions, scoped to a namespace.
- **ClusterRole**: a set of permissions, cluster-wide.
- **RoleBinding**: assigns a Role (or a ClusterRole, scoped) to a subject in a namespace.
- **ClusterRoleBinding**: assigns a ClusterRole to a subject cluster-wide.

For multi-tenancy:

- Most permissions should be namespace-scoped (Roles + RoleBindings, or ClusterRoles bound via RoleBindings).
- ClusterRoleBindings should be rare and treated as elevated permissions.
- A common pattern: a *namespace template* contains the set of standard Roles and RoleBindings for a new tenant, applied via GitOps when a tenant is provisioned.

### A worked RBAC pattern

We define three roles for typical ML platform tenants:

```yaml
# Read-only role for a team's viewers
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: tenant-viewer
  namespace: tenant-team-alpha
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log", "services", "configmaps"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["batch"]
    resources: ["jobs"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch"]

---
# Standard developer role for a team's engineers
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: tenant-developer
  namespace: tenant-team-alpha
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log", "pods/exec", "configmaps", "secrets"]
    verbs: ["*"]
  - apiGroups: ["batch"]
    resources: ["jobs", "cronjobs"]
    verbs: ["*"]
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets"]
    verbs: ["*"]
  # Note: explicitly does NOT include resourcequotas, limitranges, networkpolicies

---
# Admin role for the team's lead
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: tenant-admin
  namespace: tenant-team-alpha
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]
  # Even tenant-admins should NOT be able to edit the namespace's
  # ResourceQuota; that's the platform team's prerogative.
```

The pattern: progressively more permissive roles, but even the most permissive one is *still scoped to one namespace*. There is no cross-tenant access except through platform APIs that the platform team controls.

### Service accounts and workload identity

When a tenant's pod talks to AWS S3 (to read training data), how does AWS know it's the right tenant?

Two common patterns:

- **Long-lived secret**: an AWS access key is stored as a Kubernetes Secret in the tenant's namespace and mounted into the pod. *Avoid this.* Long-lived secrets are an audit nightmare and a rotation problem.
- **Workload identity** (AWS IRSA, GCP Workload Identity, Azure Workload Identity): the pod's Kubernetes ServiceAccount is mapped to a cloud IAM role via OIDC. The pod gets short-lived credentials. *Use this.*

We discuss this in detail in Module 09 (Security). For now, internalize: **workload identity is the modern pattern**; static secrets are the legacy pattern.

---

## Cost Allocation and Chargeback

If a platform team can't answer "which tenant cost how much last month," it cannot defend its budget, cannot make pricing decisions, cannot identify abuse, and cannot give tenants incentives to use the platform efficiently.

### Allocation models

The main models, in increasing sophistication:

1. **No allocation.** Platform team eats all cost. Tenants have no incentive to use less.
2. **Allocation only** (showback). The platform reports cost per tenant, but the tenants don't actually pay. This gives tenants information without setting up internal billing flows.
3. **Chargeback.** Each tenant's cloud cost is allocated to their cost center, which is debited in the company's financial system. This is real money.
4. **Pre-paid quotas.** Each tenant pre-purchases a quota and can use up to that amount; overruns require justification or additional funding.

Most internal platforms start at level 1, move to level 2 within a year, and may or may not progress to level 3 depending on the organization's accounting maturity.

### Practical mechanics: labels and tags

Cost allocation in cloud-native ML platforms typically rests on **labels and tags**.

Kubernetes labels on pods:

```yaml
labels:
  platform.example.com/tenant: team-alpha
  platform.example.com/workload-type: training
  platform.example.com/cost-center: cc-12345
  platform.example.com/environment: prod
```

These labels are read by:

- **Kubecost** (or Opencost) which integrates with Kubernetes metrics + cloud billing.
- **Cloud cost-explorer tools** if the labels propagate to cloud resources (e.g., persistent volumes inherit labels).
- **Internal dashboards** that aggregate cost by tenant.

For cloud resources outside Kubernetes (S3 buckets, RDS instances, dedicated EC2), use cloud-native tags:

```
Tag: tenant=team-alpha
Tag: cost-center=cc-12345
Tag: managed-by=ml-platform
```

The platform team enforces a *tagging policy* — every resource the platform creates must have a tenant tag. Resources without a tenant tag are either flagged (in a report) or rejected (by an admission controller).

### A simple chargeback formula

For a given tenant, in a given month:

```
tenant_cost =
    sum_over_pods(
      pod_runtime_hours
      * pod_resource_request
      * cloud_price_per_resource_hour
    )
  + sum_over_storage(volume_size_gb * days_active * gb_day_rate)
  + sum_over_network(egress_gb * egress_rate)
  + amortized_platform_overhead   # platform team people + infra
```

The trickiest term is `amortized_platform_overhead`. The platform team itself costs money (salaries, control-plane infra, observability). How is that split? Options:

- Evenly across tenants (simple, regressive).
- Proportional to tenant's variable cost (proportional, rewards efficiency).
- Proportional to tenant's *usage* (number of jobs, etc.) (mixed).

There is no "right" answer. Pick one, document it, and revisit.

### The "free goods become abused" problem

If a platform-provided resource is free, tenants will use it without restraint. Common examples:

- Free GPU minutes for development → tenants leave Jupyter notebooks running with GPUs idle.
- Free storage → tenants accumulate years of stale model artifacts.
- Free logs → tenants log at DEBUG level forever.

The fix is *not necessarily* charging real money. Often, *showback* (just showing the cost prominently) is enough. Tenants don't want to look bad on the dashboard.

If showback isn't enough, charge — even if internally — and watch usage drop.

---

## Data Isolation

ML platforms deal with potentially-sensitive datasets. Multi-tenancy of *data* is its own concern, distinct from multi-tenancy of *compute*.

### Bucket / dataset isolation

If each tenant has its own S3 bucket (or GCS bucket, or warehouse schema), and IAM enforces that only the tenant can read/write that bucket, you have storage-layer isolation.

Patterns:

- **One bucket per tenant** with IAM policies binding the bucket to the tenant's IAM role.
- **One bucket overall** with *prefix-level* IAM (each tenant can only read/write keys under their prefix). Cheaper to set up; weaker isolation; some IAM policies are hard to express precisely at prefix granularity.
- **Cross-tenant shared buckets** for explicitly-shared datasets (e.g., a company-wide "reference data" bucket), with read-only access for all tenants.

### Feature stores

A feature store (Feast, Tecton, etc.) sits on top of a data warehouse and exposes features for training and serving. Feature stores typically have their own multi-tenancy model:

- Features are organized into *feature views* with owners.
- A feature view's access control determines who can read it.
- Some features are *shared* across tenants (company-wide entities); some are *private* (tenant-internal).

We discuss feature stores in depth in Module 06. For now: the feature store is a multi-tenancy boundary too, and its design has to match the platform's broader tenancy model.

### Sensitive data and platform-side controls

For sensitive data — PII, financial, healthcare — the platform may need to enforce *additional* controls beyond simple IAM:

- **Audit logging**: every read of sensitive data is logged with the human + workload identity that triggered it.
- **Approval workflows**: access to certain datasets requires explicit approval from a data owner.
- **De-identification at platform boundary**: the platform automatically de-identifies sensitive columns before training jobs see them.
- **DLP scanners**: data leaving the platform (e.g., into a model artifact) is scanned for inadvertent inclusion of sensitive values.

These are large topics, mostly in Module 10 (Governance).

---

## Noisy Neighbors and Fair Sharing

Even with resource quotas, tenants can affect each other's performance through *shared resources* the quota doesn't directly cover.

### The classic noisy neighbor scenarios

- **CPU on shared nodes.** Two tenant pods, both meeting their CPU *requests*, can compete for CPU time when bursting toward their *limits*. The tenant whose pod runs first gets more cache, more memory bandwidth, etc.
- **Network on shared nodes.** Two pods on the same node share a NIC. A pod doing heavy data transfer can degrade another pod's bandwidth.
- **Disk I/O on shared nodes.** Local SSD performance is shared. A pod doing heavy random I/O affects others.
- **Cache pollution.** A pod with a large working set evicts data the platform's own caches were holding.
- **Cluster autoscaler latency.** When the cluster has to add a new node, the tenant that triggered the scale-up waits. Heavy bursting tenants can starve smaller tenants of fast-scheduling.
- **API server contention.** A tenant with many controllers issuing many list/watch calls can degrade the API server for everyone.

### Mitigations

- **Pod priorities + preemption.** Use `PriorityClass` to express that production inference is more important than speculative training. Lower-priority pods can be preempted to make room for higher-priority ones.
- **Node selectors / taints.** Critical workloads land on dedicated node pools. Less-critical workloads land on shared pools.
- **Dedicated node pools per tenant.** For very large or sensitive tenants, dedicated hardware eliminates noisy-neighbor issues entirely. Expensive.
- **Per-tenant API server quotas** (a relatively new Kubernetes feature) limit how many requests a tenant's clients can issue per second.

### Fair sharing in batch schedulers

When the platform has more job demand than capacity, fairness becomes a scheduling question. Most ML platforms use one of:

- **First-come-first-served** (FIFO). Simple, easy to explain, but a tenant that submits many jobs starves others.
- **Round-robin**. Each tenant takes turns. Fair, but doesn't account for jobs being different sizes.
- **Fair-share** (dominant resource fairness, DRF, or similar). Each tenant gets an equal share of the dominant resource (e.g., GPUs). The standard "fair" choice.
- **Weighted fair-share.** Tenants have weights based on their priority or budget. Production teams may have higher weights than experimental teams.
- **Priority + fair-share**. Jobs have priorities; among same-priority jobs, fair-share applies.

Kueue (a Kubernetes-native job queueing system) and Volcano are the two open-source schedulers ML platforms most commonly use for batch fairness. We discuss them in Module 04.

### Backpressure and queues

When demand exceeds capacity, *something* has to give. The choices, roughly in order of user-friendliness:

1. **Queue with explicit visible wait time.** Best UX: users see "your job is queued, 12 jobs ahead, estimated start in 45 minutes."
2. **Queue with unknown wait time.** Worse: users see "queued" with no estimate. They check repeatedly.
3. **Reject and retry.** Bad: users see an error and have to resubmit. Their tooling has to handle the retry loop.
4. **Hard fail.** Worst: users see an error and have to wait until they think capacity is available.

Build the queue. Show the wait time. Don't make users guess.

---

## Common Failure Modes

We have already touched on most of these throughout the chapter, but they bear collecting.

### Failure 1: Cross-tenant data leak

Symptom: tenant A's pod can read tenant B's data because IAM was misconfigured, network policy was missing, or namespace isolation was incomplete.

Root cause: usually a missing or incorrect policy. Sometimes a bug in a shared library that doesn't respect tenant scoping.

Mitigation: comprehensive policy enforcement at multiple layers (IAM, NetworkPolicy, application-layer authorization). Treat "the network won't let it happen" and "the IAM won't let it happen" as independent defenses; if either is sufficient, you have defense-in-depth.

### Failure 2: GPU starvation

Symptom: a single tenant submits many large training jobs and consumes all available GPUs. Other tenants' jobs are queued indefinitely.

Root cause: missing or generous resource quotas, no fair-share scheduling.

Mitigation: per-tenant GPU quota + a fair-share scheduler. Ideally also pre-emption so production-critical retraining can interrupt speculative work.

### Failure 3: Cost runaway

Symptom: the cloud bill triples without an obvious cause. After investigation, one tenant's misconfigured CronJob has been running every minute for a month, accumulating costs.

Root cause: no per-tenant cost visibility, no anomaly alerting.

Mitigation: showback dashboards updated daily, anomaly alerts when a tenant's spend grows by more than X% month-over-month or hour-over-hour.

### Failure 4: RBAC escalation

Symptom: a tenant developer manages to make themselves a cluster-admin, or to create resources in another tenant's namespace.

Root cause: an overly-broad ClusterRole, a misconfigured RoleBinding, a controller running with too-broad permissions that the tenant can trigger.

Mitigation: least-privilege RBAC; regular auditing; admission controllers (OPA/Gatekeeper) that reject suspicious changes.

### Failure 5: Quota inheritance bugs

Symptom: a tenant's quota was raised temporarily for a project, never lowered, and they continue to consume more than their actual budget.

Root cause: quotas managed by hand, not as code.

Mitigation: GitOps for tenant configuration. Quotas live in a git-tracked manifest. Temporary increases are PRs with expiry comments. Periodic audit ensures quotas match approved budgets.

### Failure 6: Observability blindspots

Symptom: a tenant has a production incident. Their dashboards are missing or misconfigured. The platform team has to dig through logs by hand to help debug.

Root cause: observability not treated as a tenant-onboarding step. Each tenant did their own.

Mitigation: provision per-tenant dashboards / log queries / metrics scopes automatically as part of namespace setup. The "tenant onboarded" definition-of-done includes observability.

### Failure 7: The "what is my permission" black box

Symptom: a user can't do something on the platform. They don't know why. The platform team has to investigate IAM + RBAC + NetworkPolicy + admission controllers.

Root cause: no unified view of "what am I allowed to do?"

Mitigation: a `whoami` API in the platform that surfaces effective permissions in one place. Costly to build, very high-leverage.

---

## Summary

- A platform with multiple tenants must isolate them along **seven dimensions**: compute, storage, network, identity, authorization, observability, cost. Skipping any one is a silent bug.
- Categorize your platform as **soft, hard, or hostile multi-tenant**. The threat model determines how much isolation is justified.
- **Namespace-per-tenant** is the starting Kubernetes architecture. Add ResourceQuota, LimitRange, NetworkPolicy, RBAC, and ServiceAccount per namespace to make it real.
- Namespaces are *not* security boundaries by themselves. For hard isolation, add separate node pools, Pod Security Admission, possibly runtime sandboxes.
- **ResourceQuota** caps total resources per namespace. **LimitRange** caps per-pod resources. Use both.
- **NetworkPolicy** restricts pod-to-pod traffic. Start with default-deny + explicit allow lists.
- **Identity** flows from human → token → tenant → ServiceAccount → cloud IAM via workload identity. Avoid static long-lived secrets.
- **Cost allocation** starts with showback, may progress to chargeback. Built on labels and tags; the platform enforces tagging.
- **Noisy neighbors** persist even with quotas. Mitigate with priorities, dedicated node pools, and fair-share schedulers.
- Recognize the **seven failure modes** (data leak, GPU starvation, cost runaway, RBAC escalation, quota inheritance bugs, observability blindspots, permission black box) and have at least one mitigation for each.

Multi-tenancy is the heaviest topic in Module 01 because it's where most ML platform incidents originate. Module 03 (Kubernetes) and Module 09 (Security) will return to many of these topics with implementation depth.

---

## Reflection Questions

1. For a platform you've worked with (or studied), classify it as soft/hard/hostile multi-tenant. What evidence supports the classification?
2. On the seven-dimensions table, which dimensions does your hypothetical platform invest in most heavily? Are any dimensions clearly under-invested?
3. If you had to defend "we are a soft-multi-tenant platform; we do not need namespace network policies" to a security review, what arguments would you make? Now flip — argue the opposite.
4. A tenant runs out of GPU quota. They open a ticket asking for a quota increase. What information do you (as platform engineer) need to make a defensible decision?

---

## Further Reading

- **[Kubernetes documentation: Multi-Tenancy](https://kubernetes.io/docs/concepts/security/multi-tenancy/).** Official starting point. Distinguishes soft and hard tenancy clearly.
- **[Kubernetes documentation: ResourceQuota](https://kubernetes.io/docs/concepts/policy/resource-quotas/).** Reference for quota fields and semantics.
- **[Kubernetes documentation: Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/).** With CNI plugin compatibility notes.
- **[CNCF white paper: Multi-Tenancy in Kubernetes](https://github.com/kubernetes/community/tree/master/wg-multitenancy)** (working group output). For the deeper material.
- **[Kueue documentation](https://kueue.sigs.k8s.io/).** Kubernetes-native job queueing and fair sharing.
- **[Kubecost / OpenCost](https://www.opencost.io/).** Open standard for Kubernetes cost allocation.

In the next chapter, we shift from the *isolation* concerns of multi-tenancy to the *interface* concerns of API design: versioning, contracts, deprecation, and the principles that make platform APIs worth depending on.

> **Source note.** The Kueue concepts mentioned in this chapter (ClusterQueue, LocalQueue, cohorts, ResourceFlavor, fair-share weights) are described at a deliberately high level here. For the canonical mechanics — including `BorrowingLimit`, preemption policies, and how borrowing across a cohort actually works — consult the [Kueue concepts docs](https://kueue.sigs.k8s.io/docs/concepts/) before reusing this material in production design. Verified at high level against Kueue v0.9 docs, 2026-05.
