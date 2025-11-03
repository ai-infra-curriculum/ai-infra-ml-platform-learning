# Module 1 Exercises: Platform Fundamentals

This directory contains 7 hands-on exercises to reinforce your understanding of ML platform engineering fundamentals.

## Exercise Overview

| Exercise | Title | Difficulty | Duration | Key Skills |
|----------|-------|------------|----------|------------|
| 01 | Deploy Platform API | Basic | 45 min | Kubernetes, FastAPI deployment |
| 02 | Implement Resource Quotas | Intermediate | 60 min | Resource management, validation |
| 03 | Add Job Monitoring Dashboard | Intermediate | 75 min | Prometheus, Grafana, metrics |
| 04 | Multi-Tenant Isolation Testing | Intermediate | 60 min | Security, RBAC, namespace isolation |
| 05 | Implement Distributed Training | Advanced | 120 min | PyTorch DDP, multi-node training |
| 06 | Build Python SDK | Advanced | 180 min | SDK design, client libraries |
| 07 | Auto-Scaling Implementation | Advanced | 90 min | HPA, custom metrics, scaling |

## Prerequisites

Before starting these exercises, ensure you have:

- [x] Completed Module 1 lecture notes
- [x] Local Kubernetes cluster (minikube, kind, or Docker Desktop)
- [x] Python 3.11+ installed
- [x] kubectl configured and working
- [x] Docker installed and running
- [x] Basic understanding of REST APIs

## Setup Instructions

### 1. Clone Exercise Repository

```bash
cd exercises
git clone https://github.com/ai-infra-curriculum/ml-platform-exercises.git
cd ml-platform-exercises/module-01
```

### 2. Install Dependencies

```bash
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
pip install -r requirements.txt
```

### 3. Start Local Kubernetes

```bash
# Using minikube
minikube start --cpus 4 --memory 8192 --driver=docker

# OR using kind
kind create cluster --config kind-config.yaml

# Verify cluster is running
kubectl cluster-info
kubectl get nodes
```

### 4. Create Platform Namespace

```bash
kubectl create namespace ml-platform
kubectl config set-context --current --namespace=ml-platform
```

## Exercise Structure

Each exercise directory contains:

- `README.md` - Detailed instructions and learning objectives
- `starter/` - Starter code templates
- `tests/` - Automated tests to verify your solution
- `hints.md` - Hints if you get stuck (try not to peek!)
- `solution/` - Reference implementation (use only after completing)

## How to Complete Exercises

1. **Read instructions**: Start with `README.md` in each exercise directory
2. **Review starter code**: Understand the provided templates
3. **Implement solution**: Fill in TODOs and implement required functionality
4. **Run tests**: Validate your implementation with `pytest tests/`
5. **Verify manually**: Test your solution end-to-end
6. **Compare solution**: After completing, review the reference implementation

## Evaluation Criteria

Your solutions will be evaluated on:

- ✅ **Functionality**: Does it meet all requirements?
- ✅ **Code Quality**: Clean, readable, well-documented code
- ✅ **Error Handling**: Graceful handling of edge cases
- ✅ **Testing**: All tests pass
- ✅ **Best Practices**: Follows platform engineering principles

## Getting Help

If you're stuck:

1. Review the corresponding lecture notes section
2. Check `hints.md` in the exercise directory
3. Review Kubernetes and FastAPI documentation
4. Ask in the community Slack channel
5. As a last resort, peek at the solution

## Submission (Optional)

If you're completing this as part of a course:

1. Push your solutions to a personal GitHub repository
2. Include a `REFLECTION.md` with what you learned
3. Submit the repository URL via the course platform

## Tips for Success

- **Start simple**: Get basic functionality working before adding complexity
- **Test frequently**: Run tests after each major change
- **Read error messages**: Kubernetes and Python error messages are informative
- **Use kubectl describe**: Essential for debugging Kubernetes resources
- **Check logs**: `kubectl logs <pod-name>` shows application logs
- **Ask questions**: Learning is collaborative!

---

## Exercise Summaries

### Exercise 01: Deploy Platform API ⚡ BASIC

Deploy the ML platform API you studied in the lecture to your local Kubernetes cluster.

**What You'll Build:**
- Dockerized FastAPI application
- Kubernetes Deployment and Service
- Basic authentication with tenant tokens
- Health check endpoints

**Key Learning:**
- Container orchestration with Kubernetes
- Service exposure and networking
- API deployment patterns

### Exercise 02: Implement Resource Quotas 📊 INTERMEDIATE

Add resource quota enforcement to prevent tenants from exceeding their allocated resources.

**What You'll Build:**
- Quota validation logic
- Current usage tracking
- Error responses for quota violations
- Per-tenant quota configuration

**Key Learning:**
- Resource management in multi-tenant systems
- Kubernetes ResourceQuota API
- Fair resource allocation

### Exercise 03: Add Job Monitoring Dashboard 📈 INTERMEDIATE

Create a Grafana dashboard to monitor platform health and job metrics.

**What You'll Build:**
- Prometheus instrumentation in API
- Custom metrics for jobs, tenants, resources
- Grafana dashboard with multiple panels
- Alert rules for critical conditions

**Key Learning:**
- Observability in platform engineering
- Prometheus metrics collection
- Dashboard design for operators

### Exercise 04: Multi-Tenant Isolation Testing 🔒 INTERMEDIATE

Verify that tenants are properly isolated and cannot access each other's resources.

**What You'll Build:**
- Automated security tests
- Tenant isolation validation
- RBAC verification
- Penetration test scenarios

**Key Learning:**
- Security in multi-tenant systems
- Kubernetes RBAC
- Security testing methodologies

### Exercise 05: Implement Distributed Training 🚀 ADVANCED

Extend the platform to support multi-node distributed training with PyTorch DDP.

**What You'll Build:**
- Distributed training job specification
- Multi-pod Kubernetes Job creation
- PyTorch distributed environment setup
- Example distributed training script

**Key Learning:**
- Distributed ML training patterns
- PyTorch Distributed Data Parallel
- Multi-node coordination in Kubernetes

### Exercise 06: Build Python SDK 🐍 ADVANCED

Create a Python SDK that wraps the REST API for better developer experience.

**What You'll Build:**
- Client library with idiomatic Python interface
- Resource classes (Job, Deployment, Model)
- Retries, timeouts, and error handling
- Documentation and examples

**Key Learning:**
- SDK design principles
- API client best practices
- Developer experience (DX)

### Exercise 07: Auto-Scaling Implementation ⚖️ ADVANCED

Implement Horizontal Pod Autoscaling for training jobs based on queue depth.

**What You'll Build:**
- Custom metrics server
- HPA configuration
- Job queue monitoring
- Scale-up/down policies

**Key Learning:**
- Kubernetes auto-scaling
- Custom metrics for HPA
- Resource optimization

---

## Progress Tracking

Use this checklist to track your progress:

- [ ] Exercise 01: Deploy Platform API
- [ ] Exercise 02: Implement Resource Quotas
- [ ] Exercise 03: Add Job Monitoring Dashboard
- [ ] Exercise 04: Multi-Tenant Isolation Testing
- [ ] Exercise 05: Implement Distributed Training
- [ ] Exercise 06: Build Python SDK
- [ ] Exercise 07: Auto-Scaling Implementation

**Completion Goal**: Complete at least 4 exercises (including 2 advanced) to demonstrate proficiency.

---

## Additional Resources

- [Kubernetes Tutorials](https://kubernetes.io/docs/tutorials/)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [Prometheus Python Client](https://github.com/prometheus/client_python)
- [PyTorch Distributed Tutorial](https://pytorch.org/tutorials/intermediate/ddp_tutorial.html)

**Ready to start?** Begin with [Exercise 01: Deploy Platform API](./exercise-01-deploy-api/)

---

*Last updated: November 2, 2025*
