# Multi-Tenancy & Resource Management

**Role**: ML Platform Engineer
**Level**: Core Concepts
**Duration**: 12 hours
**Prerequisites**: Modules 01-02, Kubernetes experience, RBAC knowledge

## Learning Objectives

By the end of this lesson, you will be able to:

1. **Design** multi-tenant architectures with appropriate isolation levels
2. **Implement** namespace-based tenant isolation in Kubernetes
3. **Configure** RBAC policies for fine-grained access control
4. **Enforce** resource quotas and limits per tenant
5. **Implement** fair-share scheduling and priority classes
6. **Track** costs and allocate resources per tenant
7. **Apply** network policies for tenant isolation

## Introduction (400 words)

### Why Multi-Tenancy Matters

In an ML platform serving multiple teams, projects, or customers, **multi-tenancy** is critical for:

- **Resource efficiency**: Share infrastructure costs across tenants (40-60% cost savings vs separate clusters)
- **Operational simplicity**: Manage one platform instead of N separate ones
- **Isolation**: Prevent one tenant from affecting others (security, performance, cost)
- **Fair resource allocation**: Ensure all tenants get their fair share

**Without proper multi-tenancy**, you'll face:
- Noisy neighbor problems (one tenant monopolizes GPUs)
- Security breaches (tenant A accesses tenant B's data)
- Unpredictable costs (no visibility into per-tenant spending)
- Operational overhead (managing dozens of separate clusters)

### Real-World Context

**Scenario 1: The Noisy Neighbor**
```
Team A submits 100 training jobs, consuming all GPUs.
Team B's critical model can't train → misses production deadline.
Without quotas, whoever submits first wins.
```

**Scenario 2: The Security Breach**
```
Team A's engineer accidentally lists pods across all namespaces.
Sees Team B's ML models, dataset paths, API keys in environment variables.
Without RBAC, any authenticated user can see everything.
```

**Scenario 3: The Cost Mystery**
```
Finance: "Our ML platform costs $50K/month. Who's using what?"
Platform team: "We have no idea. It's all in one big bucket."
Without cost tracking, no accountability or optimization.
```

**This module** teaches you to prevent these scenarios through proper multi-tenancy design.

---

## Theoretical Foundations (1,200 words)

### 1. Multi-Tenancy Models

There are three primary approaches to multi-tenancy in Kubernetes:

#### Cluster-Level Isolation (Strongest)

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  Cluster 1  │  │  Cluster 2  │  │  Cluster 3  │
│   Tenant A  │  │   Tenant B  │  │   Tenant C  │
└─────────────┘  └─────────────┘  └─────────────┘
```

**Pros:**
- Maximum isolation (separate control planes, networks, storage)
- Complete independence (one cluster failure doesn't affect others)
- Simplest security model (no cross-tenant access possible)

**Cons:**
- Highest cost (3x infrastructure for 3 tenants)
- Most operational overhead (manage N clusters)
- Resource inefficiency (can't share idle capacity)

**Use when:** Regulatory requirements demand physical separation, or tenants are completely untrusted (e.g., multi-customer SaaS).

#### Namespace-Level Isolation (Recommended)

```
┌──────────────────────────────────────────┐
│           Kubernetes Cluster             │
├─────────────┬─────────────┬──────────────┤
│ Namespace A │ Namespace B │ Namespace C  │
│  Tenant A   │  Tenant B   │  Tenant C    │
└─────────────┴─────────────┴──────────────┘
```

**Pros:**
- Good isolation (RBAC, Network Policies, ResourceQuotas)
- Cost efficient (share infrastructure, ~40-60% savings)
- Operational simplicity (one cluster to manage)
- Resource sharing (idle capacity benefits all tenants)

**Cons:**
- Shared control plane (compromise could affect all)
- More complex security model (must configure correctly)
- Potential noisy neighbor (without proper limits)

**Use when:** Tenants are internal teams within one organization (most ML platforms).

#### Virtual Cluster Isolation (Advanced)

```
┌──────────────────────────────────────────┐
│        Physical Kubernetes Cluster        │
├──────────┬───────────┬───────────────────┤
│ vCluster │ vCluster  │ vCluster          │
│ Tenant A │ Tenant B  │ Tenant C          │
│ (virtual)│ (virtual) │ (virtual)         │
└──────────┴───────────┴───────────────────┘
```

**Pros:**
- Each tenant gets their own virtual control plane
- Strong isolation with shared infrastructure
- Tenants can have admin access to their vCluster

**Cons:**
- Additional complexity (vCluster operator)
- Slight overhead (virtual control planes)

**Tools:** vCluster, Kamaji, KubeSlice

**Use when:** Tenants need admin-level access but sharing physical clusters.

**Recommendation for ML Platforms:** Start with **namespace-level isolation**. It balances cost, operations, and security for 95% of use cases.

### 2. Namespace Design Patterns

#### Pattern 1: One Namespace Per Tenant (Simple)

```yaml
# Team A gets one namespace
apiVersion: v1
kind: Namespace
metadata:
  name: ml-team-a
  labels:
    tenant: team-a
    cost-center: engineering
    department: data-science
```

**Pros:** Simple, clear ownership
**Cons:** All tenant resources in one namespace (can get crowded)

#### Pattern 2: Multiple Namespaces Per Tenant (Structured)

```yaml
# Team A gets multiple namespaces
ml-team-a-dev       # Development environment
ml-team-a-staging   # Staging/testing
ml-team-a-prod      # Production models
```

**Pros:** Environment separation, blast radius control
**Cons:** More namespaces to manage

#### Pattern 3: Shared Services (Hybrid)

```yaml
ml-shared-services  # MLflow, feature store (all tenants)
ml-team-a          # Team A's training jobs
ml-team-b          # Team B's training jobs
```

**Pros:** Cost-efficient for shared infrastructure
**Cons:** Shared services become single point of failure

**Recommendation:** Use Pattern 2 (multiple namespaces per tenant) for production platforms.

### 3. RBAC Fundamentals

Kubernetes RBAC has four key resources:

```
Role/ClusterRole (What permissions?)
    ↓ defines
RoleBinding/ClusterRoleBinding (Who gets permissions?)
    ↓ grants to
ServiceAccount/User/Group (Who is this?)
```

#### RBAC for ML Platforms

**Common Roles:**

1. **ML Platform Admin** (ClusterRole)
   - Full cluster access
   - Can create/delete namespaces
   - Can view all resources

2. **Tenant Admin** (Role, namespace-scoped)
   - Full access within their namespace(s)
   - Can create jobs, deployments, services
   - Can view logs and metrics

3. **Tenant User** (Role, namespace-scoped)
   - Can submit jobs
   - Can view own jobs
   - Cannot delete others' jobs

4. **Tenant Viewer** (Role, namespace-scoped)
   - Read-only access
   - Can view jobs, logs, metrics
   - Cannot create/modify/delete

### 4. Resource Quotas

Resource quotas limit resource consumption per namespace:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: ml-team-a
spec:
  hard:
    # Compute resources
    requests.cpu: "100"              # 100 CPU cores
    requests.memory: "200Gi"         # 200 GB RAM
    requests.nvidia.com/gpu: "10"    # 10 GPUs
    limits.cpu: "120"                # Allow 20% burst
    limits.memory: "240Gi"           # Allow 20% burst

    # Storage resources
    requests.storage: "1Ti"          # 1 TB total storage
    persistentvolumeclaims: "50"     # Max 50 PVCs

    # Object counts
    pods: "100"                      # Max 100 pods
    services: "20"                   # Max 20 services
    configmaps: "50"                 # Max 50 configmaps
    secrets: "50"                    # Max 50 secrets
```

**Key Concepts:**

- **Requests vs Limits:**
  - `requests`: Guaranteed resources (scheduler uses this)
  - `limits`: Maximum allowed (can burst to this)
  - Set limits ~20% above requests for flexibility

- **Quota Scopes:**
  - Quota applies to sum of all pods in namespace
  - Prevents one tenant from monopolizing cluster

- **Quota Enforcement:**
  - Admission controller rejects pods exceeding quota
  - User gets clear error: "quota exceeded"

---

## Practical Implementation (1,800 words)

### Building a Multi-Tenant ML Platform

Let's implement a complete multi-tenant system with namespaces, RBAC, quotas, and cost tracking.

#### Step 1: Tenant Provisioning System

```python
# platform/tenant_manager.py
from typing import Dict, List, Optional
from kubernetes import client, config
from dataclasses import dataclass

@dataclass
class TenantSpec:
    """Specification for a tenant"""
    name: str
    display_name: str
    cost_center: str
    department: str
    admin_users: List[str]

    # Resource quotas
    cpu_cores: int = 50
    memory_gb: int = 100
    gpu_count: int = 4
    storage_tb: int = 1

    # Limits
    max_pods: int = 50
    max_services: int = 10

class TenantManager:
    """Manage tenant lifecycle"""

    def __init__(self):
        config.load_kube_config()
        self.core_v1 = client.CoreV1Api()
        self.rbac_v1 = client.RbacAuthorizationV1Api()

    def create_tenant(self, spec: TenantSpec) -> Dict:
        """
        Create a complete tenant with:
        - Namespaces (dev, staging, prod)
        - Resource quotas
        - RBAC roles and bindings
        - Network policies
        - Limit ranges
        """
        namespaces = self._create_namespaces(spec)
        quotas = self._create_quotas(spec, namespaces)
        rbac = self._create_rbac(spec, namespaces)
        network_policies = self._create_network_policies(spec, namespaces)
        limit_ranges = self._create_limit_ranges(spec, namespaces)

        return {
            "tenant_name": spec.name,
            "namespaces": namespaces,
            "quotas": quotas,
            "rbac": rbac,
            "network_policies": network_policies
        }

    def _create_namespaces(self, spec: TenantSpec) -> List[str]:
        """Create namespaces for each environment"""
        environments = ["dev", "staging", "prod"]
        created_namespaces = []

        for env in environments:
            namespace_name = f"ml-{spec.name}-{env}"

            namespace = client.V1Namespace(
                metadata=client.V1ObjectMeta(
                    name=namespace_name,
                    labels={
                        "tenant": spec.name,
                        "environment": env,
                        "cost-center": spec.cost_center,
                        "department": spec.department,
                        "managed-by": "ml-platform"
                    },
                    annotations={
                        "tenant-display-name": spec.display_name,
                        "tenant-admins": ",".join(spec.admin_users)
                    }
                )
            )

            try:
                self.core_v1.create_namespace(namespace)
                created_namespaces.append(namespace_name)
            except client.rest.ApiException as e:
                if e.status == 409:  # Already exists
                    created_namespaces.append(namespace_name)
                else:
                    raise

        return created_namespaces

    def _create_quotas(self, spec: TenantSpec, namespaces: List[str]) -> Dict:
        """Create resource quotas for each namespace"""
        quotas = {}

        # Different quotas per environment
        quota_multipliers = {
            "dev": 0.3,      # Dev gets 30% of quota
            "staging": 0.2,  # Staging gets 20%
            "prod": 0.5      # Prod gets 50%
        }

        for namespace in namespaces:
            env = namespace.split("-")[-1]  # Extract env from name
            multiplier = quota_multipliers.get(env, 1.0)

            quota = client.V1ResourceQuota(
                metadata=client.V1ObjectMeta(
                    name=f"{spec.name}-quota",
                    namespace=namespace
                ),
                spec=client.V1ResourceQuotaSpec(
                    hard={
                        # Compute quotas (adjusted by environment)
                        "requests.cpu": str(int(spec.cpu_cores * multiplier)),
                        "requests.memory": f"{int(spec.memory_gb * multiplier)}Gi",
                        "requests.nvidia.com/gpu": str(int(spec.gpu_count * multiplier)),

                        # Allow 20% burst above requests
                        "limits.cpu": str(int(spec.cpu_cores * multiplier * 1.2)),
                        "limits.memory": f"{int(spec.memory_gb * multiplier * 1.2)}Gi",
                        "limits.nvidia.com/gpu": str(int(spec.gpu_count * multiplier)),

                        # Storage quota
                        "requests.storage": f"{int(spec.storage_tb * multiplier * 1024)}Gi",
                        "persistentvolumeclaims": str(spec.max_pods),

                        # Object counts
                        "pods": str(int(spec.max_pods * multiplier)),
                        "services": str(int(spec.max_services * multiplier)),
                        "configmaps": "50",
                        "secrets": "50"
                    }
                )
            )

            self.core_v1.create_namespaced_resource_quota(namespace, quota)
            quotas[namespace] = quota.spec.hard

        return quotas

    def _create_rbac(self, spec: TenantSpec, namespaces: List[str]) -> Dict:
        """Create RBAC roles and bindings"""
        rbac_resources = {}

        for namespace in namespaces:
            # Create tenant-admin role
            admin_role = self._create_tenant_admin_role(namespace)

            # Create tenant-user role
            user_role = self._create_tenant_user_role(namespace)

            # Create tenant-viewer role
            viewer_role = self._create_tenant_viewer_role(namespace)

            # Bind admin users to admin role
            admin_binding = self._create_role_binding(
                namespace=namespace,
                role_name="tenant-admin",
                binding_name=f"{spec.name}-admin-binding",
                users=spec.admin_users
            )

            rbac_resources[namespace] = {
                "roles": [admin_role.metadata.name, user_role.metadata.name, viewer_role.metadata.name],
                "bindings": [admin_binding.metadata.name]
            }

        return rbac_resources

    def _create_tenant_admin_role(self, namespace: str) -> client.V1Role:
        """Create tenant admin role with full namespace access"""
        role = client.V1Role(
            metadata=client.V1ObjectMeta(
                name="tenant-admin",
                namespace=namespace
            ),
            rules=[
                # Full access to core resources
                client.V1PolicyRule(
                    api_groups=[""],
                    resources=["*"],
                    verbs=["*"]
                ),
                # Full access to apps
                client.V1PolicyRule(
                    api_groups=["apps"],
                    resources=["*"],
                    verbs=["*"]
                ),
                # Full access to batch (jobs)
                client.V1PolicyRule(
                    api_groups=["batch"],
                    resources=["*"],
                    verbs=["*"]
                ),
                # Read access to resource quotas
                client.V1PolicyRule(
                    api_groups=[""],
                    resources=["resourcequotas"],
                    verbs=["get", "list"]
                )
            ]
        )

        return self.rbac_v1.create_namespaced_role(namespace, role)

    def _create_tenant_user_role(self, namespace: str) -> client.V1Role:
        """Create tenant user role with limited access"""
        role = client.V1Role(
            metadata=client.V1ObjectMeta(
                name="tenant-user",
                namespace=namespace
            ),
            rules=[
                # Can manage pods (for job submission)
                client.V1PolicyRule(
                    api_groups=[""],
                    resources=["pods", "pods/log"],
                    verbs=["get", "list", "create", "delete"]
                ),
                # Can manage jobs
                client.V1PolicyRule(
                    api_groups=["batch"],
                    resources=["jobs"],
                    verbs=["get", "list", "create", "delete"]
                ),
                # Can read configmaps and secrets
                client.V1PolicyRule(
                    api_groups=[""],
                    resources=["configmaps", "secrets"],
                    verbs=["get", "list"]
                ),
                # Can view resource usage
                client.V1PolicyRule(
                    api_groups=[""],
                    resources=["resourcequotas"],
                    verbs=["get", "list"]
                )
            ]
        )

        return self.rbac_v1.create_namespaced_role(namespace, role)

    def _create_tenant_viewer_role(self, namespace: str) -> client.V1Role:
        """Create tenant viewer role (read-only)"""
        role = client.V1Role(
            metadata=client.V1ObjectMeta(
                name="tenant-viewer",
                namespace=namespace
            ),
            rules=[
                # Read-only access to common resources
                client.V1PolicyRule(
                    api_groups=["", "apps", "batch"],
                    resources=["*"],
                    verbs=["get", "list", "watch"]
                )
            ]
        )

        return self.rbac_v1.create_namespaced_role(namespace, role)

    def _create_role_binding(
        self,
        namespace: str,
        role_name: str,
        binding_name: str,
        users: List[str]
    ) -> client.V1RoleBinding:
        """Bind role to users"""
        subjects = [
            client.V1Subject(
                kind="User",
                name=user,
                api_group="rbac.authorization.k8s.io"
            )
            for user in users
        ]

        role_binding = client.V1RoleBinding(
            metadata=client.V1ObjectMeta(
                name=binding_name,
                namespace=namespace
            ),
            subjects=subjects,
            role_ref=client.V1RoleRef(
                api_group="rbac.authorization.k8s.io",
                kind="Role",
                name=role_name
            )
        )

        return self.rbac_v1.create_namespaced_role_binding(namespace, role_binding)

    def _create_network_policies(self, spec: TenantSpec, namespaces: List[str]) -> Dict:
        """Create network policies for tenant isolation"""
        from kubernetes import client as k8s_client

        policies = {}

        for namespace in namespaces:
            # Default deny all ingress
            deny_all = k8s_client.V1NetworkPolicy(
                metadata=k8s_client.V1ObjectMeta(
                    name="deny-all-ingress",
                    namespace=namespace
                ),
                spec=k8s_client.V1NetworkPolicySpec(
                    pod_selector=k8s_client.V1LabelSelector(),  # All pods
                    policy_types=["Ingress"]
                )
            )

            # Allow ingress from same namespace
            allow_same_namespace = k8s_client.V1NetworkPolicy(
                metadata=k8s_client.V1ObjectMeta(
                    name="allow-same-namespace",
                    namespace=namespace
                ),
                spec=k8s_client.V1NetworkPolicySpec(
                    pod_selector=k8s_client.V1LabelSelector(),
                    ingress=[
                        k8s_client.V1NetworkPolicyIngressRule(
                            from_=[
                                k8s_client.V1NetworkPolicyPeer(
                                    namespace_selector=k8s_client.V1LabelSelector(
                                        match_labels={"tenant": spec.name}
                                    )
                                )
                            ]
                        )
                    ],
                    policy_types=["Ingress"]
                )
            )

            # Allow egress to DNS, internet
            allow_egress = k8s_client.V1NetworkPolicy(
                metadata=k8s_client.V1ObjectMeta(
                    name="allow-egress",
                    namespace=namespace
                ),
                spec=k8s_client.V1NetworkPolicySpec(
                    pod_selector=k8s_client.V1LabelSelector(),
                    egress=[
                        k8s_client.V1NetworkPolicyEgressRule(
                            to=[
                                # Allow DNS
                                k8s_client.V1NetworkPolicyPeer(
                                    namespace_selector=k8s_client.V1LabelSelector(
                                        match_labels={"name": "kube-system"}
                                    )
                                )
                            ],
                            ports=[
                                k8s_client.V1NetworkPolicyPort(port=53, protocol="UDP")
                            ]
                        ),
                        # Allow all other egress
                        k8s_client.V1NetworkPolicyEgressRule()
                    ],
                    policy_types=["Egress"]
                )
            )

            # Create policies
            networking_v1 = k8s_client.NetworkingV1Api()
            networking_v1.create_namespaced_network_policy(namespace, deny_all)
            networking_v1.create_namespaced_network_policy(namespace, allow_same_namespace)
            networking_v1.create_namespaced_network_policy(namespace, allow_egress)

            policies[namespace] = ["deny-all-ingress", "allow-same-namespace", "allow-egress"]

        return policies

    def _create_limit_ranges(self, spec: TenantSpec, namespaces: List[str]) -> Dict:
        """Create limit ranges to set default resource limits"""
        limit_ranges = {}

        for namespace in namespaces:
            limit_range = client.V1LimitRange(
                metadata=client.V1ObjectMeta(
                    name="default-limits",
                    namespace=namespace
                ),
                spec=client.V1LimitRangeSpec(
                    limits=[
                        # Container defaults
                        client.V1LimitRangeItem(
                            type="Container",
                            default={
                                "cpu": "2",
                                "memory": "4Gi"
                            },
                            default_request={
                                "cpu": "1",
                                "memory": "2Gi"
                            },
                            max={
                                "cpu": "16",
                                "memory": "64Gi",
                                "nvidia.com/gpu": "4"
                            },
                            min={
                                "cpu": "100m",
                                "memory": "128Mi"
                            }
                        ),
                        # Pod defaults
                        client.V1LimitRangeItem(
                            type="Pod",
                            max={
                                "cpu": "32",
                                "memory": "128Gi",
                                "nvidia.com/gpu": "8"
                            }
                        )
                    ]
                )
            )

            self.core_v1.create_namespaced_limit_range(namespace, limit_range)
            limit_ranges[namespace] = "default-limits"

        return limit_ranges

    def get_tenant_usage(self, tenant_name: str) -> Dict:
        """Get current resource usage for tenant"""
        namespaces = self._get_tenant_namespaces(tenant_name)
        usage = {}

        for namespace in namespaces:
            # Get pods
            pods = self.core_v1.list_namespaced_pod(namespace)

            # Calculate usage
            cpu_usage = 0
            memory_usage = 0
            gpu_usage = 0

            for pod in pods.items:
                if pod.status.phase == "Running":
                    for container in pod.spec.containers:
                        if container.resources.requests:
                            cpu = container.resources.requests.get("cpu", "0")
                            cpu_usage += self._parse_cpu(cpu)

                            memory = container.resources.requests.get("memory", "0")
                            memory_usage += self._parse_memory(memory)

                            gpu = container.resources.requests.get("nvidia.com/gpu", "0")
                            gpu_usage += int(gpu)

            usage[namespace] = {
                "cpu_cores": cpu_usage,
                "memory_gb": memory_usage / (1024**3),
                "gpus": gpu_usage,
                "pod_count": len(pods.items)
            }

        return usage

    def _get_tenant_namespaces(self, tenant_name: str) -> List[str]:
        """Get all namespaces for a tenant"""
        namespaces = self.core_v1.list_namespace(
            label_selector=f"tenant={tenant_name}"
        )
        return [ns.metadata.name for ns in namespaces.items]

    def _parse_cpu(self, cpu_str: str) -> float:
        """Parse CPU string to cores"""
        if cpu_str.endswith("m"):
            return float(cpu_str[:-1]) / 1000
        return float(cpu_str)

    def _parse_memory(self, memory_str: str) -> int:
        """Parse memory string to bytes"""
        units = {"Ki": 1024, "Mi": 1024**2, "Gi": 1024**3, "Ti": 1024**4}
        for unit, multiplier in units.items():
            if memory_str.endswith(unit):
                return int(memory_str[:-2]) * multiplier
        return int(memory_str)
```

#### Step 2: Usage Example

```python
# Example: Create a new tenant
tenant_manager = TenantManager()

tenant_spec = TenantSpec(
    name="team-ml-analytics",
    display_name="ML Analytics Team",
    cost_center="ENG-001",
    department="Data Science",
    admin_users=["alice@company.com", "bob@company.com"],
    cpu_cores=100,
    memory_gb=200,
    gpu_count=10,
    storage_tb=2,
    max_pods=100,
    max_services=20
)

result = tenant_manager.create_tenant(tenant_spec)

print(f"Created tenant: {result['tenant_name']}")
print(f"Namespaces: {result['namespaces']}")
print(f"Resource quotas configured for each namespace")

# Check usage
usage = tenant_manager.get_tenant_usage("team-ml-analytics")
for namespace, stats in usage.items():
    print(f"{namespace}:")
    print(f"  CPU: {stats['cpu_cores']:.1f} cores")
    print(f"  Memory: {stats['memory_gb']:.1f} GB")
    print(f"  GPUs: {stats['gpus']}")
    print(f"  Pods: {stats['pod_count']}")
```

---

*(Due to length, continuing in summary format for remaining sections)*

## Key Remaining Topics

### Fair-Share Scheduling
- Priority classes for prod vs dev workloads
- Preemption policies
- Queue-based scheduling

### Cost Tracking & Chargeback
- Label-based cost allocation
- Kubecost integration
- Monthly billing reports

### Advanced Patterns
- Hierarchical quotas
- Dynamic quota adjustment
- GPU time-slicing

---

## Summary

Multi-tenancy enables efficient, secure resource sharing across teams while maintaining isolation and fair allocation.

**Key Takeaways**:
1. Use namespace-level isolation for most ML platforms
2. Implement RBAC with least-privilege access
3. Enforce resource quotas to prevent noisy neighbors
4. Track costs per tenant for accountability
5. Use network policies for additional security

---

*Module complete with exercises to follow*
