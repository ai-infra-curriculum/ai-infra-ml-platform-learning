# Module 01: Platform Fundamentals

**Duration**: 8 hours
**Level**: Foundations
**Prerequisites**: Strong Python, Kubernetes basics, API design experience

## Overview

This module introduces the foundational concepts of ML platform engineering. You'll learn what distinguishes platform engineering from traditional infrastructure work, understand the core components of modern ML platforms, and explore architectural patterns for building scalable, multi-tenant systems.

## Learning Objectives

By completing this module, you will be able to:

1. ✅ Define what an ML platform is and articulate its value proposition
2. ✅ Differentiate between ML platform engineering, ML engineering, and infrastructure engineering
3. ✅ Identify the core components of a modern ML platform architecture
4. ✅ Explain multi-tenancy patterns and their importance in ML platforms
5. ✅ Design API-first interfaces for ML platform services
6. ✅ Evaluate platform abstractions and their impact on developer experience
7. ✅ Apply platform thinking principles to ML infrastructure problems

## Module Structure

### Lecture Notes

- **[01-introduction-to-ml-platforms.md](./lecture-notes/01-introduction-to-ml-platforms.md)** (4,000+ words)
  - What is an ML Platform?
  - Platform vs Infrastructure Engineering
  - Multi-tenancy architecture patterns
  - API design principles
  - Platform thinking and abstractions
  - Practical implementation with FastAPI + Kubernetes
  - Real-world case studies (Uber, Netflix, Airbnb)

### Exercises

- **[7 Hands-On Exercises](./exercises/)**
  - 1 Basic: Deploy Platform API (45 min)
  - 3 Intermediate: Resource Quotas, Monitoring, Security (60-75 min each)
  - 3 Advanced: Distributed Training, SDK, Auto-scaling (90-180 min each)

### Resources

- **[Additional Resources](./resources/README.md)**
  - Official documentation links
  - Open-source ML platform projects
  - Books and articles
  - Video tutorials

## Key Topics Covered

### 1. Platform Engineering Fundamentals (2 hours)

- What makes platform engineering unique
- Platform-as-a-product mindset
- Internal developer platforms (IDPs)
- Developer experience (DX) principles

### 2. ML Platform Architecture (2 hours)

- 7 layers of an ML platform
- Component interactions
- Technology stack choices
- Scalability patterns

### 3. Multi-Tenancy Design (1.5 hours)

- Namespace-level isolation
- Resource quotas and limits
- RBAC and policy enforcement
- Cost allocation strategies

### 4. API Design Patterns (1.5 hours)

- Progressive disclosure
- Sensible defaults
- Convention over configuration
- Batteries included, but swappable

### 5. Practical Implementation (1 hour)

- Building a training job API
- Kubernetes resource management
- Authentication and authorization
- Deployment patterns

## Time Allocation

| Activity | Duration |
|----------|----------|
| Lecture Notes | 3 hours |
| Exercises (4/7) | 4 hours |
| Quizzes | 30 min |
| Additional Reading | 30 min |
| **Total** | **8 hours** |

## Prerequisites Check

Before starting, ensure you have:

- [x] **Python 3.11+** with strong programming skills (3+ years)
- [x] **Kubernetes fundamentals**: pods, deployments, services, namespaces
- [x] **API design experience**: REST APIs, request/response patterns
- [x] **Understanding of ML workflows**: training, inference, model serving
- [x] **Docker basics**: building images, running containers
- [x] **Linux/Unix CLI**: comfortable with bash, ssh, basic sysadmin

## Getting Started

### Step 1: Read Lecture Notes

Start with the comprehensive lecture notes:

```bash
cd lecture-notes
cat 01-introduction-to-ml-platforms.md
```

Or open in your favorite Markdown viewer for better readability.

### Step 2: Complete Exercises

Work through at least 4 exercises (including 2 advanced):

```bash
cd exercises
# Read the exercise overview
cat README.md

# Start with Exercise 01
cd exercise-01-deploy-api
cat README.md
```

### Step 3: Take Assessment

Complete the knowledge check and practical challenge at the end of the lecture notes.

### Step 4: Explore Resources

Review additional resources for deeper understanding:

```bash
cd resources
cat README.md
```

## Success Criteria

To successfully complete this module:

- ✅ Score 80%+ on knowledge check (10 questions)
- ✅ Complete 4/7 exercises (including 2 advanced)
- ✅ Pass practical challenge (extend platform with model deployment)
- ✅ Demonstrate understanding of platform thinking principles

## What's Next?

After completing this module, proceed to:

**[Module 02: API Design for ML Platforms](../mod-002-api-platforms/)**
- RESTful vs gRPC trade-offs
- API versioning strategies
- SDK design patterns
- OpenAPI specification
- Client code generation

## Common Questions

**Q: Can I skip exercises if I already know Kubernetes?**
A: While you may be familiar with Kubernetes, the exercises focus on ML-specific patterns and platform abstractions. We recommend completing at least the advanced exercises.

**Q: How much time should I spend on this module?**
A: Plan for 8-10 hours total. Don't rush—understanding these foundations is critical for later modules.

**Q: What if I get stuck on exercises?**
A: Check hints.md in each exercise directory, review lecture notes, or ask in the community Slack. Solutions are provided as a last resort.

**Q: Do I need a cloud account?**
A: No, all exercises run on a local Kubernetes cluster (minikube/kind). Cloud deployment is optional.

## Additional Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [FastAPI Tutorial](https://fastapi.tiangolo.com/tutorial/)
- [Building ML Platforms - Uber Engineering](https://www.uber.com/blog/michelangelo-machine-learning-platform/)
- [Metaflow Documentation](https://docs.metaflow.org/)

## Feedback

Found an issue or have suggestions? Please open an issue on GitHub or reach out on Slack.

---

**Module Status**: ✅ Content Complete
**Last Updated**: November 2, 2025
**Next Module**: [API Design for ML Platforms →](../mod-002-api-platforms/)
