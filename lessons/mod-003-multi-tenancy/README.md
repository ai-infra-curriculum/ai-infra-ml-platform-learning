# Module 03: Multi-Tenancy & Resource Management

**Duration**: 12 hours
**Level**: Core Concepts
**Prerequisites**: Module 02 completed, Kubernetes fundamentals, RBAC concepts

## Overview

Master multi-tenancy patterns and resource management for ML platforms, implementing namespace isolation, RBAC policies, resource quotas, and cost tracking systems.

## Learning Objectives

By the end of this module, you will be able to:

1. ✅ Design multi-tenant architectures using Kubernetes namespaces
2. ✅ Implement RBAC policies for tenant isolation
3. ✅ Configure resource quotas and LimitRanges
4. ✅ Enforce network policies for traffic isolation
5. ✅ Build cost tracking and chargeback systems
6. ✅ Implement fair-share scheduling with PriorityClasses
7. ✅ Evaluate trade-offs between isolation models (cluster, namespace, vCluster)

## Module Structure

### Lecture Notes
- **[01-multi-tenancy-resource-management.md](./lecture-notes/01-multi-tenancy-resource-management.md)** (4,500+ words)
  - Multi-tenancy models and trade-offs
  - Namespace design patterns
  - RBAC implementation (roles, bindings)
  - Resource quota strategies
  - Network isolation with NetworkPolicies
  - Cost allocation and chargeback
  - Fair-share scheduling
  - Production case studies

### Exercises
- **[6 Hands-On Exercises](./exercises/)** (~9.75 hours total)
  - Exercise 01: Automate Tenant Provisioning (90 min)
  - Exercise 02: Implement RBAC Policies (75 min)
  - Exercise 03: Configure Resource Quotas (60 min)
  - Exercise 04: Enforce Network Isolation (120 min) - Advanced
  - Exercise 05: Build Cost Tracking System (150 min) - Advanced
  - Exercise 06: Implement Fair-Share Scheduling (90 min)

## Key Topics Covered

### 1. Multi-Tenancy Models
- **Cluster-level isolation**: Separate clusters per tenant
- **Namespace-level isolation**: Shared cluster, isolated namespaces (recommended)
- **Virtual clusters**: vCluster, Kamaji for improved isolation
- Cost-benefit analysis: 40-60% savings with namespace isolation

### 2. Resource Management
- **ResourceQuotas**: CPU, memory, GPU, storage limits
- **LimitRanges**: Default resource requests/limits
- **Environment distribution**: Dev (30%), Staging (20%), Prod (50%)
- **Quota enforcement**: Preventing resource contention

### 3. RBAC Security
- **Tenant roles**: Admin, User, Viewer (namespace-scoped)
- **Platform roles**: Platform Admin, Platform Operator (cluster-scoped)
- **PolicyRules**: Fine-grained permission control
- **RoleBindings**: User/group to role mappings

### 4. Network Isolation
- **Default deny** policies for security
- **Intra-namespace** communication
- **Cross-environment** access control
- **Platform services** allowlist (DNS, monitoring, logging)
- **NetworkPolicies**: Ingress and egress rules

### 5. Cost Tracking & Chargeback
- **Resource usage metrics**: CPU-hours, memory GB-hours, GPU-hours
- **Cost calculation**: Configurable rates per resource type
- **Chargeback reports**: Daily, weekly, monthly invoices
- **Budget alerts**: Notifications when approaching limits
- **Prometheus integration**: Real-time metrics collection

### 6. Fair-Share Scheduling
- **PriorityClasses**: Workload prioritization (high/medium/low/best-effort)
- **Preemption**: Lower-priority pod eviction for critical workloads
- **Queue management**: Fair allocation among tenants
- **GPU scheduling**: Time-slicing and sharing strategies

## Code Examples

This module includes production-ready implementations:

- **TenantManager class** (~500 lines): Complete tenant provisioning automation
- **RBAC generator**: Creates roles and bindings for all tenant levels
- **ResourceQuota calculator**: Environment-specific allocation
- **NetworkPolicy templates**: Default deny + selective allow patterns
- **CostTracker service**: Prometheus-based usage tracking and billing
- **PriorityClass definitions**: 4-tier priority system

## Real-World Applications

### Multi-Tenancy at Scale
- **Uber**: 100+ ML teams on shared platform
- **Netflix**: Namespace isolation for 50+ engineering teams
- **Airbnb**: Resource quotas preventing "noisy neighbor" issues

### Resource Management
- **Cost savings**: 40-60% reduction vs per-tenant clusters
- **Utilization**: 70-85% average cluster utilization (vs 30-40% dedicated)
- **Isolation**: 99.9% cross-tenant traffic blocked with NetworkPolicies

### Chargeback Systems
- **Transparency**: Teams see real resource costs
- **Accountability**: Budget limits drive efficient resource usage
- **Planning**: Historical data informs capacity planning

## Success Criteria

After completing this module, you should be able to:

- [ ] Provision complete multi-tenant environments in <30 seconds
- [ ] Implement RBAC policies that pass security audits
- [ ] Configure resource quotas that prevent over-allocation
- [ ] Block 100% of unauthorized cross-tenant traffic
- [ ] Calculate tenant costs with ±5% accuracy
- [ ] Ensure production workloads always scheduled (high priority)

## Tools and Technologies

**Kubernetes APIs**:
- CoreV1Api (namespaces, quotas, limitranges)
- RbacAuthorizationV1Api (roles, rolebindings)
- NetworkingV1Api (networkpolicies)
- SchedulingV1Api (priorityclasses)

**Python Libraries**:
- `kubernetes` - Official Kubernetes Python client
- `pydantic` - Data validation for tenant specs
- `prometheus-api-client` - Metrics collection
- `pandas` - Report generation

**Platform Components**:
- Kubernetes 1.25+ (for stable NetworkPolicy APIs)
- Prometheus (resource metrics)
- PostgreSQL (cost data storage)

## Prerequisites Checklist

Before starting this module, ensure you have:

- [x] Completed Module 01 (Platform Fundamentals)
- [x] Completed Module 02 (API Design)
- [x] Access to Kubernetes cluster (kubectl configured)
- [x] Understanding of Kubernetes resources (pods, deployments, services)
- [x] Basic knowledge of RBAC concepts
- [x] Python 3.9+ with kubernetes client installed
- [x] (Optional) Prometheus for Exercise 05

## Estimated Time

| Component | Duration |
|-----------|----------|
| Lecture reading | 1.5 hours |
| Exercises (4 minimum) | 6-10 hours |
| Practice and review | 1-2 hours |
| **Total** | **8.5-13.5 hours** |

**Recommended pace**: Complete over 3-4 days, allowing time to absorb concepts and practice.

## Assessment

### Knowledge Check (10 questions)
Located in lecture notes - tests understanding of:
- Multi-tenancy models
- RBAC design
- Resource quota calculation
- Network policy rules
- Cost allocation strategies

### Practical Challenge
Build complete multi-tenant platform with:
- 3 tenants (dev, staging, prod per tenant)
- Full RBAC hierarchy
- Resource quotas enforced
- Network isolation verified
- Cost tracking operational

## What's Next?

**[Module 04: Feature Store Architecture](../mod-004-feature-store/)**

After mastering multi-tenancy, you'll learn how to build feature stores for ML platforms - the foundation for consistent feature engineering across training and inference.

---

## Additional Resources

### Documentation
- [Kubernetes Multi-Tenancy Working Group](https://github.com/kubernetes-sigs/multi-tenancy)
- [Namespace Best Practices](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
- [vCluster Documentation](https://www.vcluster.com/docs/)
- [Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)

### Articles & Papers
- ["Multi-Tenancy in Kubernetes" (CNCF)](https://www.cncf.io/blog/2020/07/21/multi-tenancy-in-kubernetes/)
- ["Resource Management in Kubernetes"](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- ["Cost Optimization Strategies" (AWS EKS Best Practices)](https://aws.github.io/aws-eks-best-practices/cost_optimization/cost_opt_compute/)

### Tools
- [Kyverno](https://kyverno.io/) - Policy management for Kubernetes
- [OPA Gatekeeper](https://open-policy-agent.github.io/gatekeeper/) - Policy enforcement
- [Kube-bench](https://github.com/aquasecurity/kube-bench) - CIS Kubernetes benchmark checks
- [Kubecost](https://www.kubecost.com/) - Cost monitoring and optimization

### Videos
- [KubeCon: Multi-Tenancy Deep Dive](https://www.youtube.com/results?search_query=kubecon+multi-tenancy)
- [Kubernetes Security Best Practices](https://www.youtube.com/results?search_query=kubernetes+security+rbac)

---

**Status**: ✅ Complete | **Last Updated**: November 2, 2025 | **Version**: 1.0

**Feedback**: Found an issue or have suggestions? Please open an issue in the repository.
