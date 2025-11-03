# Module 3 Exercises: Multi-Tenancy & Resource Management

This directory contains 6 hands-on exercises to master multi-tenancy patterns and resource management in ML platforms.

## Exercise Overview

| Exercise | Title | Difficulty | Duration | Key Skills |
|----------|-------|------------|----------|------------|
| 01 | Automate Tenant Provisioning | Intermediate | 90 min | Kubernetes API, namespace creation, automation |
| 02 | Implement RBAC Policies | Intermediate | 75 min | RBAC design, roles, bindings, security |
| 03 | Configure Resource Quotas | Basic | 60 min | ResourceQuotas, LimitRanges, resource management |
| 04 | Enforce Network Isolation | Advanced | 120 min | NetworkPolicies, DNS, service mesh, security |
| 05 | Build Cost Tracking System | Advanced | 150 min | Resource monitoring, cost allocation, chargeback |
| 06 | Implement Fair-Share Scheduling | Intermediate | 90 min | Kubernetes scheduler, priority classes, preemption |

## Prerequisites

- [x] Completed Module 2 exercises
- [x] Kubernetes cluster access (kubectl configured)
- [x] Understanding of Kubernetes resources (namespaces, pods, deployments)
- [x] Python 3.9+ with kubernetes client library
- [x] Basic understanding of RBAC concepts

**Total Time**: 585 minutes (~9.75 hours)

---

## Exercise 01: Automate Tenant Provisioning

**Objective**: Build an automated tenant provisioning system that creates namespaces, quotas, and RBAC resources for new ML teams.

**Difficulty**: Intermediate | **Duration**: 90 minutes

### Requirements

Create a `TenantProvisioner` class that:
1. Accepts tenant specifications (name, resource limits, environments)
2. Creates 3 namespaces per tenant (dev, staging, prod)
3. Applies resource quotas based on environment
4. Sets up default RBAC roles (admin, user, viewer)
5. Configures network policies for isolation
6. Returns tenant credentials and access information

### Implementation Steps

1. **Design tenant specification schema** (15 min)
   - Define Pydantic model for `TenantSpec`
   - Include fields: name, team, cpu_cores, memory_gb, gpu_count, environments

2. **Implement namespace creation** (20 min)
   - Use Kubernetes API to create namespaces
   - Add labels for tenant identification
   - Add annotations for metadata (team, cost-center)

3. **Apply resource quotas** (25 min)
   - Create ResourceQuota objects for each namespace
   - Use environment multipliers (30% dev, 20% staging, 50% prod)
   - Include CPU, memory, GPU, and pod limits

4. **Configure RBAC** (20 min)
   - Create 3 roles per namespace (admin, user, viewer)
   - Create RoleBindings for initial users
   - Document permission matrix

5. **Test provisioning workflow** (10 min)
   - Provision 2 test tenants
   - Verify namespace creation
   - Validate resource quotas
   - Test RBAC permissions

### Starter Code

```python
from pydantic import BaseModel, Field
from typing import List, Dict, Optional
from kubernetes import client, config
import logging

class TenantSpec(BaseModel):
    """Specification for tenant creation"""
    name: str = Field(..., pattern=r'^[a-z0-9-]+$')
    team: str
    cost_center: str
    cpu_cores: int = Field(..., ge=1, le=100)
    memory_gb: int = Field(..., ge=4, le=512)
    gpu_count: int = Field(default=0, ge=0, le=16)
    max_pods: int = Field(default=100, ge=10, le=500)
    environments: List[str] = Field(default=["dev", "staging", "prod"])
    admins: List[str] = Field(default=[])

class TenantProvisioner:
    """Automate tenant provisioning"""

    def __init__(self):
        config.load_kube_config()
        self.core_api = client.CoreV1Api()
        self.rbac_api = client.RbacAuthorizationV1Api()
        self.logger = logging.getLogger(__name__)

    def provision_tenant(self, spec: TenantSpec) -> Dict:
        """
        Provision complete tenant infrastructure.

        Returns:
            Dict with namespace names, quotas, and access info
        """
        # TODO: Implement tenant provisioning
        pass
```

### Success Criteria

- [ ] Tenant provisioning completes in <30 seconds
- [ ] All namespaces created with correct labels
- [ ] Resource quotas properly configured
- [ ] RBAC roles and bindings created
- [ ] Tenant can deploy workloads in dev namespace
- [ ] Tenant cannot exceed resource quotas

### Validation

```bash
# Verify namespaces
kubectl get ns -l tenant=acme-ml

# Check resource quotas
kubectl get resourcequota -n acme-ml-dev
kubectl describe resourcequota acme-ml-quota -n acme-ml-dev

# Verify RBAC
kubectl get roles,rolebindings -n acme-ml-dev

# Test as tenant user
kubectl auth can-i create pods --as=john@acme.com -n acme-ml-dev
kubectl auth can-i delete deployments --as=john@acme.com -n acme-ml-prod
```

---

## Exercise 02: Implement RBAC Policies

**Objective**: Design and implement fine-grained RBAC policies for multi-tenant ML platform.

**Difficulty**: Intermediate | **Duration**: 75 minutes

### Requirements

Create a comprehensive RBAC system with:
1. **3 tenant-level roles**: admin, user, viewer
2. **2 platform-level roles**: platform-admin, platform-operator
3. **Custom permissions** for ML workloads (training jobs, model serving)
4. **Namespace-scoped** bindings for tenant isolation
5. **Cluster-scoped** roles for platform administrators

### Role Definitions

**Tenant Admin** (namespace-scoped):
- Full access to all resources in tenant namespaces
- Can create/delete pods, deployments, services
- Can view secrets (but not cross-namespace)
- Can modify resource quotas (within limits)

**Tenant User** (namespace-scoped):
- Create/update ML workloads (jobs, deployments)
- View logs and exec into pods
- Cannot delete production resources
- Cannot modify resource quotas

**Tenant Viewer** (namespace-scoped):
- Read-only access to all resources
- Can view logs but not exec into pods
- Cannot create/modify resources

**Platform Admin** (cluster-scoped):
- Full cluster access
- Can create/delete tenants
- Can modify global policies

**Platform Operator** (cluster-scoped):
- Read-only cluster access
- Can view all tenant resources
- Can troubleshoot issues
- Cannot modify tenant resources

### Implementation Steps

1. **Design permission matrix** (15 min)
   - Document all resource types and verbs
   - Map roles to permissions

2. **Implement tenant roles** (25 min)
   - Create Role resources for each tenant role
   - Use PolicyRules to define permissions
   - Test with `kubectl auth can-i`

3. **Implement platform roles** (20 min)
   - Create ClusterRole resources
   - Define aggregation labels for composability

4. **Create role bindings** (10 min)
   - RoleBindings for tenant users
   - ClusterRoleBindings for platform admins

5. **Test RBAC enforcement** (5 min)
   - Verify each role's permissions
   - Test negative cases (denials)

### Starter Code

```python
from kubernetes import client
from typing import List, Dict

def create_tenant_admin_role(namespace: str) -> client.V1Role:
    """Create tenant-admin role with full namespace access"""
    return client.V1Role(
        metadata=client.V1ObjectMeta(
            name="tenant-admin",
            namespace=namespace,
            labels={"rbac.ml-platform/role": "tenant-admin"}
        ),
        rules=[
            client.V1PolicyRule(
                api_groups=["", "apps", "batch"],
                resources=["*"],
                verbs=["*"]
            ),
            # TODO: Add more rules
        ]
    )

def create_role_binding(
    namespace: str,
    role_name: str,
    subjects: List[str]
) -> client.V1RoleBinding:
    """Bind role to users/groups"""
    # TODO: Implement role binding
    pass
```

### Permission Matrix

| Resource | Tenant Admin | Tenant User | Tenant Viewer | Platform Admin | Platform Operator |
|----------|--------------|-------------|---------------|----------------|-------------------|
| Pods | CRUD | CRU | R | CRUD | R |
| Deployments | CRUD | CRU | R | CRUD | R |
| Jobs | CRUD | CRUD | R | CRUD | R |
| Secrets | CRUD | R (own) | - | CRUD | R |
| ConfigMaps | CRUD | CRU | R | CRUD | R |
| ResourceQuotas | RU | R | R | CRUD | R |
| Namespaces | - | - | - | CRUD | R |

**Legend**: C=Create, R=Read, U=Update, D=Delete

### Success Criteria

- [ ] All 5 roles created with correct permissions
- [ ] Role bindings correctly scope access
- [ ] Tenant users isolated to their namespaces
- [ ] Platform admins have cluster-wide access
- [ ] Negative tests confirm denials work

---

## Exercise 03: Configure Resource Quotas

**Objective**: Implement resource quotas to prevent resource contention and ensure fair allocation.

**Difficulty**: Basic | **Duration**: 60 minutes

### Requirements

Configure ResourceQuotas and LimitRanges for:
1. **CPU and memory** limits per namespace
2. **GPU allocation** (if available)
3. **Pod count** limits
4. **Storage** quotas (PVC size and count)
5. **Default resource requests/limits** via LimitRanges

### Resource Allocation Strategy

**Per-Tenant Base Allocation**:
- CPU: 16 cores
- Memory: 64 GB
- GPU: 2 GPUs
- Max Pods: 100

**Environment Distribution**:
- Dev: 30% of base
- Staging: 20% of base
- Prod: 50% of base

### Implementation Steps

1. **Create ResourceQuota manifests** (20 min)
2. **Define LimitRanges** (15 min)
3. **Apply quotas to test namespaces** (10 min)
4. **Test quota enforcement** (15 min)

### Starter Code

```yaml
# resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-quota
  namespace: acme-ml-dev
spec:
  hard:
    requests.cpu: "5"           # 16 * 0.3 = 4.8 ≈ 5
    requests.memory: "20Gi"     # 64 * 0.3 = 19.2 ≈ 20
    requests.nvidia.com/gpu: "1" # 2 * 0.3 = 0.6 ≈ 1
    limits.cpu: "6"             # 5 * 1.2 burst
    limits.memory: "24Gi"
    persistentvolumeclaims: "10"
    requests.storage: "100Gi"
    pods: "30"                  # 100 * 0.3

---
# limit-range.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: tenant-limits
  namespace: acme-ml-dev
spec:
  limits:
  - max:
      cpu: "4"
      memory: "16Gi"
    min:
      cpu: "100m"
      memory: "128Mi"
    default:
      cpu: "500m"
      memory: "1Gi"
    defaultRequest:
      cpu: "250m"
      memory: "512Mi"
    type: Container
  - max:
      cpu: "8"
      memory: "32Gi"
    min:
      cpu: "100m"
      memory: "128Mi"
    type: Pod
```

### Python Implementation

```python
def create_resource_quota(
    namespace: str,
    cpu_cores: int,
    memory_gb: int,
    gpu_count: int,
    multiplier: float
) -> client.V1ResourceQuota:
    """Create ResourceQuota for namespace"""
    # TODO: Implement quota creation
    pass

def create_limit_range(namespace: str) -> client.V1LimitRange:
    """Create LimitRange for default resource constraints"""
    # TODO: Implement LimitRange creation
    pass
```

### Testing

```bash
# Deploy pod without resource requests (should get defaults)
kubectl run test-pod --image=nginx -n acme-ml-dev

# Verify defaults were applied
kubectl get pod test-pod -n acme-ml-dev -o yaml | grep -A 10 resources

# Try to exceed quota
kubectl run big-pod --image=nginx --requests=cpu=10 -n acme-ml-dev
# Should fail with quota exceeded error

# Check quota usage
kubectl get resourcequota tenant-quota -n acme-ml-dev
kubectl describe resourcequota tenant-quota -n acme-ml-dev
```

### Success Criteria

- [ ] ResourceQuotas applied to all tenant namespaces
- [ ] LimitRanges provide sensible defaults
- [ ] Pods without requests/limits get defaults
- [ ] Quota enforcement prevents over-allocation
- [ ] Quota status shows current usage

---

## Exercise 04: Enforce Network Isolation

**Objective**: Implement network policies to isolate tenant traffic and prevent cross-tenant access.

**Difficulty**: Advanced | **Duration**: 120 minutes

### Requirements

Implement network isolation using:
1. **Default deny** all ingress/egress
2. **Allow intra-namespace** communication
3. **Allow cross-environment** communication (dev → staging → prod)
4. **Deny cross-tenant** communication
5. **Allow platform services** (DNS, monitoring, logging)

### Network Policy Architecture

```
┌─────────────────────────────────────────┐
│         Tenant A (acme-ml)              │
│  ┌─────────┬─────────┬─────────┐       │
│  │   Dev   │ Staging │  Prod   │       │
│  │  ✓ ←→   │   ✓ ←→  │   ✓     │       │
│  └─────────┴─────────┴─────────┘       │
│         ↓ (allowed)                     │
│    Platform Services                    │
│    (DNS, monitoring)                    │
└─────────────────────────────────────────┘
                 ✗ (denied)
┌─────────────────────────────────────────┐
│         Tenant B (beta-ml)              │
│  ┌─────────┬─────────┬─────────┐       │
│  │   Dev   │ Staging │  Prod   │       │
│  └─────────┴─────────┴─────────┘       │
└─────────────────────────────────────────┘
```

### Implementation Steps

1. **Design network policy strategy** (20 min)
   - Document traffic flows
   - Identify platform services
   - Define policy precedence

2. **Implement default deny policies** (15 min)
   - Deny all ingress by default
   - Deny all egress by default

3. **Allow intra-namespace traffic** (20 min)
   - Pods can communicate within same namespace
   - Use namespace selectors

4. **Allow DNS and platform services** (25 min)
   - Allow egress to kube-dns
   - Allow egress to monitoring (Prometheus)
   - Allow egress to logging (Fluentd)

5. **Implement cross-environment policies** (20 min)
   - Dev can access staging
   - Staging can access prod
   - Prod cannot access dev/staging

6. **Test network isolation** (20 min)
   - Deploy test pods in multiple namespaces
   - Verify allowed traffic works
   - Verify denied traffic blocked

### Starter Code

```yaml
# default-deny.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: acme-ml-dev
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
# allow-intra-namespace.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-intra-namespace
  namespace: acme-ml-dev
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector: {}
  egress:
  - to:
    - podSelector: {}

---
# allow-dns.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: acme-ml-dev
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    - podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
```

### Python Implementation

```python
def create_default_deny_policy(namespace: str) -> client.V1NetworkPolicy:
    """Create default deny all policy"""
    # TODO: Implement
    pass

def create_allow_dns_policy(namespace: str) -> client.V1NetworkPolicy:
    """Allow DNS queries"""
    # TODO: Implement
    pass

def create_cross_environment_policy(
    source_ns: str,
    target_ns: str
) -> client.V1NetworkPolicy:
    """Allow traffic from dev to staging"""
    # TODO: Implement
    pass
```

### Testing Network Policies

```bash
# Deploy test pods
kubectl run test-dev --image=nicolaka/netshoot -n acme-ml-dev -- sleep 3600
kubectl run test-staging --image=nicolaka/netshoot -n acme-ml-staging -- sleep 3600
kubectl run test-other --image=nicolaka/netshoot -n beta-ml-dev -- sleep 3600

# Test intra-namespace (should work)
kubectl exec test-dev -n acme-ml-dev -- ping -c 3 <pod-ip-in-same-ns>

# Test cross-namespace same tenant (should work if allowed)
kubectl exec test-dev -n acme-ml-dev -- curl http://<service-in-staging>

# Test cross-tenant (should fail)
kubectl exec test-dev -n acme-ml-dev -- curl http://<service-in-beta-ml>

# Test DNS (should work)
kubectl exec test-dev -n acme-ml-dev -- nslookup kubernetes.default

# Check network policies
kubectl get networkpolicies -A
kubectl describe networkpolicy default-deny-all -n acme-ml-dev
```

### Success Criteria

- [ ] Default deny policies block all traffic
- [ ] Intra-namespace communication works
- [ ] Cross-tenant communication blocked
- [ ] DNS queries succeed
- [ ] Platform services accessible
- [ ] Monitoring works (Prometheus can scrape)

---

## Exercise 05: Build Cost Tracking System

**Objective**: Implement a cost tracking and chargeback system for multi-tenant resource usage.

**Difficulty**: Advanced | **Duration**: 150 minutes

### Requirements

Build a system that:
1. **Tracks resource usage** per tenant (CPU, memory, GPU hours)
2. **Calculates costs** based on configurable rates
3. **Generates chargeback reports** (daily, weekly, monthly)
4. **Exports to CSV** for accounting systems
5. **Provides API** for querying costs
6. **Sends alerts** when tenants approach budget limits

### Cost Model

```python
# Resource rates (per hour)
RATES = {
    "cpu_core": 0.05,      # $0.05/core/hour
    "memory_gb": 0.01,     # $0.01/GB/hour
    "gpu": 2.50,           # $2.50/GPU/hour
    "storage_gb": 0.10,    # $0.10/GB/month
}
```

### Architecture

```
┌────────────────┐
│   Prometheus   │ ← scrapes resource metrics
└────────┬───────┘
         │
         ↓ queries
┌────────────────┐
│ Cost Tracker   │ ← calculates costs
│   Service      │
└────────┬───────┘
         │
         ↓ stores
┌────────────────┐
│   PostgreSQL   │ ← cost data
└────────┬───────┘
         │
         ↓ exports
┌────────────────┐
│  Chargeback    │ ← reports
│    Reports     │
└────────────────┘
```

### Implementation Steps

1. **Set up Prometheus queries** (30 min)
   - Query container CPU usage
   - Query container memory usage
   - Query GPU utilization (if available)
   - Aggregate by namespace/tenant

2. **Implement cost calculation** (40 min)
   - Fetch metrics from Prometheus
   - Apply cost rates
   - Calculate hourly costs
   - Store in database

3. **Build reporting system** (40 min)
   - Generate daily summaries
   - Create monthly invoices
   - Export to CSV
   - Send via email/Slack

4. **Create cost API** (30 min)
   - FastAPI endpoints for cost queries
   - Filter by tenant, date range
   - Aggregation functions

5. **Test cost tracking** (10 min)
   - Deploy workloads in test tenant
   - Verify cost calculation
   - Generate test reports

### Starter Code

```python
from prometheus_api_client import PrometheusConnect
from datetime import datetime, timedelta
from typing import Dict, List
import pandas as pd

# Cost rates (per hour)
RATES = {
    "cpu_core": 0.05,
    "memory_gb": 0.01,
    "gpu": 2.50,
}

class CostTracker:
    """Track and calculate tenant costs"""

    def __init__(self, prometheus_url: str):
        self.prom = PrometheusConnect(url=prometheus_url, disable_ssl=True)

    def get_cpu_usage(
        self,
        namespace: str,
        start: datetime,
        end: datetime
    ) -> float:
        """
        Get CPU core-hours for namespace.

        Returns:
            Total CPU core-hours
        """
        query = f'''
            sum(
                rate(container_cpu_usage_seconds_total{{
                    namespace="{namespace}",
                    container!=""
                }}[5m])
            ) * (
                (timestamp() - {start.timestamp()})
            ) / 3600
        '''
        # TODO: Execute query and calculate total
        pass

    def get_memory_usage(
        self,
        namespace: str,
        start: datetime,
        end: datetime
    ) -> float:
        """Get memory GB-hours for namespace"""
        query = f'''
            sum(
                container_memory_usage_bytes{{
                    namespace="{namespace}",
                    container!=""
                }} / 1024 / 1024 / 1024
            ) * (
                (timestamp() - {start.timestamp()})
            ) / 3600
        '''
        # TODO: Execute query
        pass

    def calculate_costs(
        self,
        namespace: str,
        start: datetime,
        end: datetime
    ) -> Dict:
        """
        Calculate total costs for namespace.

        Returns:
            Dict with breakdown: {
                'cpu_hours': float,
                'cpu_cost': float,
                'memory_gb_hours': float,
                'memory_cost': float,
                'gpu_hours': float,
                'gpu_cost': float,
                'total_cost': float
            }
        """
        cpu_hours = self.get_cpu_usage(namespace, start, end)
        memory_gb_hours = self.get_memory_usage(namespace, start, end)

        cpu_cost = cpu_hours * RATES["cpu_core"]
        memory_cost = memory_gb_hours * RATES["memory_gb"]

        return {
            "cpu_hours": cpu_hours,
            "cpu_cost": cpu_cost,
            "memory_gb_hours": memory_gb_hours,
            "memory_cost": memory_cost,
            "total_cost": cpu_cost + memory_cost
        }

    def generate_invoice(
        self,
        tenant: str,
        namespaces: List[str],
        month: int,
        year: int
    ) -> pd.DataFrame:
        """Generate monthly invoice for tenant"""
        # TODO: Implement invoice generation
        pass

# FastAPI cost API
from fastapi import FastAPI, Query
from pydantic import BaseModel

app = FastAPI()

class CostResponse(BaseModel):
    tenant: str
    namespace: str
    start_date: datetime
    end_date: datetime
    cpu_cost: float
    memory_cost: float
    gpu_cost: float
    total_cost: float

@app.get("/api/v1/costs/{tenant}", response_model=List[CostResponse])
async def get_tenant_costs(
    tenant: str,
    start_date: datetime = Query(...),
    end_date: datetime = Query(...)
):
    """Get cost breakdown for tenant"""
    # TODO: Implement cost query
    pass
```

### Prometheus Queries Reference

```promql
# CPU usage (cores) - rate over 5 minutes
sum by (namespace) (
    rate(container_cpu_usage_seconds_total{
        container!="",
        namespace!="kube-system"
    }[5m])
)

# Memory usage (GB)
sum by (namespace) (
    container_memory_usage_bytes{
        container!="",
        namespace!="kube-system"
    } / 1024 / 1024 / 1024
)

# GPU usage (count)
sum by (namespace) (
    nvidia_gpu_duty_cycle{namespace!="kube-system"} / 100
)

# Storage usage (GB)
sum by (namespace, persistentvolumeclaim) (
    kubelet_volume_stats_used_bytes / 1024 / 1024 / 1024
)
```

### Report Example

```
┌──────────────────────────────────────────────────┐
│         Monthly Cost Report - October 2025       │
│         Tenant: acme-ml                          │
└──────────────────────────────────────────────────┘

Environment    CPU Hours    Memory GB-Hrs    GPU Hours    Total Cost
────────────────────────────────────────────────────────────────────
Dev               1,250          3,200            24        $249.70
Staging             840          2,100            16        $183.10
Prod              3,600          9,500           120        $695.00
────────────────────────────────────────────────────────────────────
Total             5,690         14,800           160      $1,127.80

Breakdown:
  - CPU:    5,690 hours × $0.05 = $284.50
  - Memory: 14,800 GB-hrs × $0.01 = $148.00
  - GPU:    160 hours × $2.50 = $400.00
  - Storage: 2,500 GB × $0.10 = $250.00 (prorated)

Budget: $1,500.00
Usage: 75.2%
Remaining: $372.20
```

### Success Criteria

- [ ] Cost tracker queries Prometheus successfully
- [ ] Cost calculations accurate (±5%)
- [ ] Reports generated automatically
- [ ] CSV export works for accounting
- [ ] API returns cost data correctly
- [ ] Alerts sent when approaching budget

---

## Exercise 06: Implement Fair-Share Scheduling

**Objective**: Configure Kubernetes scheduler for fair resource allocation among tenants.

**Difficulty**: Intermediate | **Duration**: 90 minutes

### Requirements

Implement fair-share scheduling using:
1. **PriorityClasses** for workload prioritization
2. **Preemption** to handle resource contention
3. **Pod priority** to ensure critical workloads run
4. **Fair-share queue** for batch jobs
5. **GPU scheduling** with time-slicing (if available)

### Priority Hierarchy

```
High Priority (1000):  Production inference services
                       ↓
Medium Priority (500): Staging workloads, important batch jobs
                       ↓
Low Priority (100):    Dev workloads, experiments
                       ↓
Best-Effort (0):       Pre-emptible research jobs
```

### Implementation Steps

1. **Create PriorityClasses** (20 min)
   - Define 4 priority levels
   - Configure preemption policies

2. **Implement pod priorities** (25 min)
   - Add priorityClassName to deployments
   - Test preemption behavior

3. **Configure fair-share scheduler** (30 min)
   - Install scheduler plugins (if needed)
   - Configure queue weights

4. **Test scheduling decisions** (15 min)
   - Deploy workloads with different priorities
   - Verify preemption occurs
   - Check fair-share allocation

### Starter Code

```yaml
# priority-classes.yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: production-high
value: 1000
globalDefault: false
description: "High priority for production inference services"
preemptionPolicy: PreemptLowerPriority

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: production-medium
value: 500
globalDefault: false
description: "Medium priority for staging and important batch jobs"
preemptionPolicy: PreemptLowerPriority

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: development-low
value: 100
globalDefault: true  # Default for all pods
description: "Low priority for dev and experiments"
preemptionPolicy: PreemptLowerPriority

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: research-best-effort
value: 0
globalDefault: false
description: "Best-effort for pre-emptible research jobs"
preemptionPolicy: Never
```

### Usage Example

```yaml
# high-priority-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prod-inference
  namespace: acme-ml-prod
spec:
  replicas: 3
  selector:
    matchLabels:
      app: inference
  template:
    metadata:
      labels:
        app: inference
    spec:
      priorityClassName: production-high  # High priority
      containers:
      - name: model-server
        image: tensorflow/serving:latest
        resources:
          requests:
            cpu: "2"
            memory: "4Gi"
            nvidia.com/gpu: "1"
          limits:
            cpu: "4"
            memory: "8Gi"
            nvidia.com/gpu: "1"
```

### Testing Preemption

```python
"""Test preemption behavior"""

def test_preemption():
    """
    1. Fill cluster with low-priority pods
    2. Submit high-priority pod
    3. Verify low-priority pod is preempted
    """
    # Create low-priority deployment
    low_priority = create_deployment(
        name="low-priority-workload",
        replicas=10,
        priority_class="development-low",
        resources={"cpu": "1", "memory": "2Gi"}
    )

    # Wait for all pods to be running
    wait_for_pods_running(low_priority)

    # Create high-priority deployment (should trigger preemption)
    high_priority = create_deployment(
        name="high-priority-workload",
        replicas=5,
        priority_class="production-high",
        resources={"cpu": "2", "memory": "4Gi"}
    )

    # Verify preemption occurred
    assert get_pod_count(high_priority, status="Running") == 5
    assert get_pod_count(low_priority, status="Pending") > 0
```

### Verification

```bash
# Apply priority classes
kubectl apply -f priority-classes.yaml

# Check priority classes
kubectl get priorityclasses

# Deploy test workloads
kubectl apply -f low-priority-deployment.yaml
kubectl apply -f high-priority-deployment.yaml

# Watch preemption
kubectl get pods -A -w

# Check pod priority
kubectl get pod <pod-name> -o yaml | grep -A 5 priority

# View preemption events
kubectl get events --sort-by='.lastTimestamp' | grep -i preempt
```

### Success Criteria

- [ ] 4 PriorityClasses created
- [ ] High-priority pods preempt low-priority pods
- [ ] Production workloads never starved
- [ ] Fair allocation among same-priority pods
- [ ] Preemption events logged correctly

---

## Progress Tracking

- [ ] Exercise 01: Automate Tenant Provisioning
- [ ] Exercise 02: Implement RBAC Policies
- [ ] Exercise 03: Configure Resource Quotas
- [ ] Exercise 04: Enforce Network Isolation
- [ ] Exercise 05: Build Cost Tracking System
- [ ] Exercise 06: Implement Fair-Share Scheduling

**Completion Goal**: Complete at least 4 exercises (including 1 advanced).

---

## Additional Resources

### Documentation
- [Kubernetes RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Pod Priority and Preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)

### Tools
- [kubectl](https://kubernetes.io/docs/reference/kubectl/) - Kubernetes CLI
- [Prometheus](https://prometheus.io/) - Metrics and monitoring
- [k9s](https://k9scli.io/) - Terminal UI for Kubernetes
- [kubectx/kubens](https://github.com/ahmetb/kubectx) - Context and namespace switching

### Libraries
- [kubernetes-client/python](https://github.com/kubernetes-client/python) - Python Kubernetes client
- [prometheus-api-client](https://github.com/4n4nd/prometheus-api-client-python) - Prometheus Python client
- [pydantic](https://docs.pydantic.dev/) - Data validation

---

*Last updated: November 2, 2025*
