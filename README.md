# ML Platform Engineer - Learning Repository

![License](https://img.shields.io/badge/license-MIT-blue.svg)
![Python](https://img.shields.io/badge/python-3.11+-blue.svg)
![Kubernetes](https://img.shields.io/badge/kubernetes-1.28+-blue.svg)
![Status](https://img.shields.io/badge/status-active-success.svg)

> **A comprehensive, hands-on curriculum for aspiring ML Platform Engineers**

This repository provides a complete learning path for becoming an ML Platform Engineer, focusing on building self-service ML platforms, feature stores, workflow orchestration systems, and developer tooling that enable data scientists and ML engineers to be productive at scale.

## Table of Contents

- [About This Repository](#about-this-repository)
- [Who Is This For?](#who-is-this-for)
- [Learning Path Overview](#learning-path-overview)
- [Prerequisites](#prerequisites)
- [Repository Structure](#repository-structure)
- [Projects Overview](#projects-overview)
- [Getting Started](#getting-started)
- [Learning Modules](#learning-modules)
- [How to Use This Repository](#how-to-use-this-repository)
- [Assessment & Certification](#assessment--certification)
- [Support & Community](#support--community)
- [Contributing](#contributing)
- [License](#license)
- [Acknowledgments](#acknowledgments)

## About This Repository

This is the **learning repository** for the ML Platform Engineer career track. It contains:

- **Comprehensive curriculum** covering platform engineering principles for ML systems
- **5 major hands-on projects** with detailed requirements and architecture guides
- **8-10 structured learning modules** with lectures, exercises, and resources
- **Code stubs and templates** to guide your implementation
- **Assessments and practical exams** to validate your skills
- **Real-world scenarios** based on industry best practices

This repository focuses on **learning and practice**. For complete reference implementations, see the companion [ai-infra-ml-platform-solutions](https://github.com/ai-infra-curriculum/ai-infra-ml-platform-solutions) repository.

## Who Is This For?

This curriculum is designed for:

### Target Audience

- **Senior AI Infrastructure Engineers** looking to specialize in platform engineering
- **Backend Engineers** transitioning into ML infrastructure
- **DevOps Engineers** wanting to build ML-specific platforms
- **ML Engineers** seeking to understand platform internals
- **Platform Engineers** from other domains entering ML space

### Required Background

You should have:

- **Strong Python programming** (3+ years experience)
- **Kubernetes fundamentals** (deployment, services, operators)
- **API design experience** (REST, gRPC)
- **Database knowledge** (SQL and NoSQL)
- **Linux/Unix proficiency**
- **CI/CD experience** (GitHub Actions, GitLab CI, Jenkins)
- **Cloud platform experience** (AWS, GCP, or Azure)

### Recommended Experience

Helpful but not required:

- Experience with ML frameworks (PyTorch, TensorFlow)
- Understanding of ML training and deployment workflows
- Familiarity with microservices architectures
- Background in distributed systems
- Knowledge of infrastructure as code (Terraform, Pulumi)

## Learning Path Overview

This curriculum follows a progressive learning path:

```
┌─────────────────────────────────────────────────────────────────┐
│                     ML Platform Engineer                        │
│                    (Level 2.5A - Advanced)                      │
└─────────────────────────────────────────────────────────────────┘
                                │
                    ┌───────────┴───────────┐
                    │                       │
            ┌───────▼────────┐     ┌───────▼────────┐
            │   Foundation   │     │   Core Skills  │
            │   (Weeks 1-2)  │     │   (Weeks 3-6)  │
            └───────┬────────┘     └───────┬────────┘
                    │                       │
                    └───────────┬───────────┘
                                │
                    ┌───────────▼────────────┐
                    │   Hands-On Projects    │
                    │   (Weeks 7-24)         │
                    │   ├── Project 1: 4 wks │
                    │   ├── Project 2: 4 wks │
                    │   ├── Project 3: 4 wks │
                    │   ├── Project 4: 4 wks │
                    │   └── Project 5: 4 wks │
                    └───────────┬────────────┘
                                │
                    ┌───────────▼────────────┐
                    │   Assessment &         │
                    │   Certification        │
                    │   (Weeks 25-26)        │
                    └────────────────────────┘
```

### Estimated Timeline

- **Total Duration**: 6-8 months (part-time, 15-20 hours/week)
- **Full-Time**: 3-4 months (40 hours/week)
- **Project Hours**: 600-700 hours total
- **Theory & Lectures**: 80-100 hours
- **Assessments**: 20-30 hours

## Prerequisites

### Technical Prerequisites

Before starting this curriculum, ensure you have:

#### 1. Programming & Development
- [ ] Python 3.11+ development environment
- [ ] Git version control proficiency
- [ ] Unix/Linux command line expertise
- [ ] Docker and containerization knowledge
- [ ] Understanding of microservices architecture

#### 2. Infrastructure & Operations
- [ ] Kubernetes cluster access (local or cloud)
- [ ] kubectl and Helm proficiency
- [ ] Cloud platform account (AWS, GCP, or Azure)
- [ ] Infrastructure as Code basics (Terraform or Pulumi)
- [ ] CI/CD pipeline experience

#### 3. Data & ML Basics
- [ ] Basic understanding of ML workflows (training, inference)
- [ ] Familiarity with ML frameworks (PyTorch or TensorFlow)
- [ ] Understanding of model serving concepts
- [ ] Basic data engineering knowledge

#### 4. Software Engineering
- [ ] RESTful API design principles
- [ ] Database design (PostgreSQL, MongoDB)
- [ ] Authentication & authorization (OAuth, RBAC)
- [ ] Testing practices (unit, integration, e2e)

### Environment Setup

See [CURRICULUM.md](./CURRICULUM.md) for detailed environment setup instructions.

## Repository Structure

```
ai-infra-ml-platform-learning/
├── README.md                          # This file
├── CURRICULUM.md                      # Detailed curriculum and learning guide
├── LICENSE                            # MIT License
├── CODE_OF_CONDUCT.md                 # Community guidelines
├── CONTRIBUTING.md                    # Contribution guidelines
├── requirements.txt                   # Python dependencies
├── .gitignore                         # Git ignore patterns
│
├── .github/                           # GitHub configuration
│   ├── workflows/                     # CI/CD workflows
│   │   ├── validate-code.yml          # Code validation
│   │   └── test-stubs.yml             # Test stub validation
│   ├── ISSUE_TEMPLATE/                # Issue templates
│   │   ├── bug_report.md
│   │   ├── feature_request.md
│   │   ├── question.md
│   │   └── project_help.md
│   └── PULL_REQUEST_TEMPLATE.md       # PR template
│
├── lessons/                           # Learning modules
│   ├── module-01-platform-fundamentals/
│   ├── module-02-api-design/
│   ├── module-03-multi-tenancy/
│   ├── module-04-feature-stores/
│   ├── module-05-workflow-orchestration/
│   ├── module-06-model-management/
│   ├── module-07-developer-experience/
│   ├── module-08-observability/
│   └── module-09-security-governance/
│
├── projects/                          # Hands-on projects
│   ├── project-01-platform-core/      # Self-service ML platform
│   ├── project-02-feature-store/      # Enterprise feature store
│   ├── project-03-workflow-orchestration/  # ML workflow engine
│   ├── project-04-model-registry/     # Model management system
│   └── project-05-developer-portal/   # Developer portal & SDK
│
├── assessments/                       # Knowledge assessments
│   ├── quizzes/                       # Module quizzes
│   ├── practical-exams/               # Hands-on exams
│   └── capstone/                      # Final capstone project
│
├── resources/                         # Additional resources
│   ├── reading-list.md                # Books, papers, articles
│   ├── tools.md                       # Required tools & setup
│   ├── references.md                  # API docs, standards
│   ├── glossary.md                    # Terminology
│   └── cheat-sheets/                  # Quick reference guides
│
├── progress/                          # Track your progress
│   ├── progress-tracker.md            # Personal progress log
│   ├── skill-matrix.md                # Skills assessment
│   └── portfolio-guide.md             # Building your portfolio
│
└── community/                         # Community resources
    ├── FAQ.md                         # Frequently asked questions
    ├── discussions.md                 # Discussion links
    └── showcase.md                    # Student projects showcase
```

## Projects Overview

This curriculum includes **5 comprehensive projects** that build upon each other to create a complete ML platform ecosystem:

### Project 1: Self-Service ML Platform Core
**Duration**: 4 weeks | **Difficulty**: Advanced | **Hours**: 120

Build the foundational platform that enables data scientists to provision compute resources, submit training jobs, and deploy models without direct infrastructure access.

**Key Concepts**:
- Multi-tenancy architecture
- Kubernetes operators and CRDs
- RESTful and gRPC API design
- Resource quota management
- Authentication & authorization (RBAC, SSO)
- Platform observability

**Technologies**: Python, FastAPI, Kubernetes, PostgreSQL, Redis, Prometheus, Grafana

**Deliverables**:
- Platform API service
- Kubernetes operator for ML workloads
- Multi-tenant resource management
- User & team management system
- Comprehensive API documentation

### Project 2: Enterprise Feature Store Implementation
**Duration**: 4 weeks | **Difficulty**: Advanced | **Hours**: 120

Build a production-grade feature store using Feast as foundation, extended with real-time serving, versioning, lineage tracking, and monitoring capabilities.

**Key Concepts**:
- Online vs offline feature stores
- Point-in-time correct retrieval
- Feature versioning and lineage
- Real-time feature serving
- Feature drift detection
- Data consistency in distributed systems

**Technologies**: Python, Feast, Redis, S3, Kafka, Apache Spark, PostgreSQL

**Deliverables**:
- Feature registry service
- Online feature store (Redis)
- Offline feature store (S3/Parquet)
- Feature transformation SDK
- Monitoring & drift detection

### Project 3: ML Workflow Orchestration Platform
**Duration**: 4 weeks | **Difficulty**: Advanced | **Hours**: 120

Build a comprehensive workflow orchestration system for defining, scheduling, and managing complex ML pipelines as code.

**Key Concepts**:
- DAG-based workflow execution
- Task dependency resolution
- Distributed task execution
- Pipeline versioning and lineage
- Event-driven orchestration
- Retry and error handling

**Technologies**: Python, Apache Airflow (or custom), Kubernetes, PostgreSQL, Redis, Celery

**Deliverables**:
- Workflow definition SDK
- DAG scheduler and executor
- Task queue management
- Workflow monitoring UI
- Integration with platform core

### Project 4: Model Registry & Management
**Duration**: 4 weeks | **Difficulty**: Advanced | **Hours**: 120

Build a centralized model registry for versioning, metadata management, lifecycle tracking, and governance of ML models.

**Key Concepts**:
- Model versioning strategies
- Artifact storage and retrieval
- Model lineage tracking
- Model promotion workflows
- A/B testing support
- Model governance policies

**Technologies**: Python, MLflow (extended), S3, PostgreSQL, Kubernetes

**Deliverables**:
- Model registry service
- Model metadata management
- Version control system
- Deployment workflow engine
- Model performance tracking

### Project 5: Developer Portal & SDK
**Duration**: 4 weeks | **Difficulty**: Advanced | **Hours**: 120

Build a comprehensive developer portal with documentation, Python SDK, CLI tools, and interactive tutorials that make the platform accessible to ML practitioners.

**Key Concepts**:
- SDK design patterns
- API client generation
- Interactive documentation
- Developer experience (DX)
- Platform adoption metrics
- Self-service onboarding

**Technologies**: Python, React, TypeScript, Backstage (optional), OpenAPI, Docusaurus

**Deliverables**:
- Python SDK for platform
- CLI tool for operations
- Developer portal website
- Interactive tutorials
- API playground

## Getting Started

### Step 1: Environment Setup

1. **Clone this repository**:
   ```bash
   git clone https://github.com/ai-infra-curriculum/ai-infra-ml-platform-learning.git
   cd ai-infra-ml-platform-learning
   ```

2. **Install dependencies**:
   ```bash
   python -m venv venv
   source venv/bin/activate  # On Windows: venv\Scripts\activate
   pip install -r requirements.txt
   ```

3. **Setup Kubernetes cluster** (choose one):
   ```bash
   # Option 1: Local (minikube)
   minikube start --cpus 4 --memory 8192 --driver=docker

   # Option 2: Local (kind)
   kind create cluster --config resources/kind-config.yaml

   # Option 3: Cloud (GKE example)
   gcloud container clusters create ml-platform-learning \
     --zone us-central1-a --num-nodes 3 --machine-type n1-standard-4
   ```

4. **Verify setup**:
   ```bash
   python scripts/verify-setup.py
   ```

### Step 2: Start Learning

1. **Review the curriculum**:
   ```bash
   # Read the comprehensive curriculum guide
   cat CURRICULUM.md
   ```

2. **Begin with Module 01**:
   ```bash
   cd lessons/module-01-platform-fundamentals
   cat README.md
   ```

3. **Work through modules sequentially**:
   - Read lecture notes
   - Complete exercises
   - Build hands-on labs
   - Take module quizzes

### Step 3: Start Your First Project

1. **Navigate to Project 1**:
   ```bash
   cd projects/project-01-platform-core
   cat README.md
   ```

2. **Review project requirements**:
   ```bash
   cat requirements.md
   cat architecture.md
   ```

3. **Begin implementation**:
   - Follow the TODO comments in code stubs
   - Run tests frequently: `pytest tests/`
   - Refer to lecture notes when stuck
   - Check the FAQ for common issues

### Step 4: Track Your Progress

1. **Update progress tracker**:
   ```bash
   vim progress/progress-tracker.md
   ```

2. **Complete skill assessments**:
   ```bash
   cd assessments/quizzes
   python module-01-quiz.py
   ```

3. **Build your portfolio**:
   - Document your implementations
   - Create architecture diagrams
   - Write blog posts about learnings
   - Share code on GitHub

## Learning Modules

### Module 01: Platform Fundamentals (Week 1)
Introduction to ML platform engineering, platform thinking, multi-tenancy patterns, and API-first design.

**Topics**:
- What is an ML Platform?
- Platform vs Infrastructure Engineering
- Multi-tenancy architecture patterns
- API design principles
- Platform thinking and abstractions

**Duration**: 8 hours

### Module 02: API Design for ML Platforms (Week 2)
RESTful and gRPC API design, versioning, documentation, and SDK development.

**Topics**:
- RESTful API design for ML workloads
- gRPC for high-performance operations
- API versioning strategies
- OpenAPI specification
- SDK design patterns

**Duration**: 10 hours

### Module 03: Multi-Tenancy & Resource Management (Week 3)
Implementing secure multi-tenancy, resource quotas, isolation, and fair-share scheduling.

**Topics**:
- Kubernetes namespace isolation
- Resource quotas and limits
- RBAC and policy enforcement
- Cost allocation and chargeback
- Priority-based scheduling

**Duration**: 12 hours

### Module 04: Feature Store Architecture (Week 4)
Understanding feature stores, online/offline storage, point-in-time correctness, and feature serving.

**Topics**:
- Feature store concepts
- Online vs offline stores
- Point-in-time correct retrieval
- Feature versioning
- Real-time feature serving

**Duration**: 12 hours

### Module 05: Workflow Orchestration (Week 5)
DAG-based workflows, task dependency management, distributed execution, and monitoring.

**Topics**:
- DAG concepts and design
- Task dependency resolution
- Distributed task execution
- Workflow versioning
- Monitoring and alerting

**Duration**: 12 hours

### Module 06: Model Management & Registry (Week 6)
Model versioning, metadata management, lineage tracking, and deployment workflows.

**Topics**:
- Model versioning strategies
- Metadata and artifact storage
- Lineage and provenance tracking
- Model promotion workflows
- Governance policies

**Duration**: 10 hours

### Module 07: Developer Experience & Tooling (Week 7)
Building SDKs, CLI tools, documentation, and creating exceptional developer experiences.

**Topics**:
- SDK design principles
- CLI tool development
- Interactive documentation
- Developer onboarding
- Platform adoption strategies

**Duration**: 10 hours

### Module 08: Observability & Monitoring (Week 8)
Platform metrics, logging, tracing, alerting, and building observable ML systems.

**Topics**:
- Metrics collection (Prometheus)
- Distributed tracing (Jaeger)
- Log aggregation (ELK)
- Alerting strategies
- SLIs and SLOs for ML platforms

**Duration**: 12 hours

### Module 09: Security & Governance (Week 9)
Authentication, authorization, data privacy, compliance, and audit logging.

**Topics**:
- Authentication mechanisms (SSO, SAML, OIDC)
- RBAC and policy enforcement
- Data privacy and encryption
- Compliance (GDPR, HIPAA)
- Audit logging

**Duration**: 10 hours

## How to Use This Repository

### For Self-Paced Learners

1. **Follow the structured path**: Complete modules in order, as each builds on previous knowledge
2. **Hands-on practice**: Code along with exercises, don't just read
3. **Complete all projects**: Each project reinforces critical skills
4. **Take assessments**: Validate your understanding with quizzes and exams
5. **Join the community**: Ask questions, share learnings, help others

### For Bootcamps & Instructors

This curriculum is designed for:
- **University courses** (semester-long)
- **Bootcamp programs** (12-16 weeks intensive)
- **Corporate training** (upskilling programs)
- **Study groups** (team learning)

**Instructor resources**:
- Lecture notes per module (under `lessons/<module>/lecture-notes/`)
- Assessment answer keys (in the [solutions repo](https://github.com/ai-infra-curriculum/ai-infra-ml-platform-solutions))
- Module exercises ready to assign as homework
- Lecture slides are not maintained here today — file a Discussion if you want to contribute a deck

### For Hiring Managers

Use this curriculum to:
- **Assess candidates**: Projects demonstrate real-world skills
- **Onboard new hires**: Structured ramp-up for ML platform roles
- **Create internal training**: Customize for your stack
- **Evaluate skills**: Practical exams show competency

## Assessment & Certification

### Module Assessments

Each module includes:
- **Quiz** (10-15 questions): Test conceptual understanding
- **Practical exercise**: Hands-on coding challenge
- **Passing score**: 80% required to proceed

### Project Assessments

Each project includes:
- **Functional requirements checklist**: All features implemented
- **Code quality review**: Clean, tested, documented code
- **Architecture review**: Sound design decisions
- **Performance benchmarks**: Meets performance criteria
- **Documentation review**: Comprehensive docs

### Capstone Project

Final assessment combining all skills:
- **Design a complete ML platform** from scratch
- **Present architecture** to peer review panel
- **Implement core components**
- **Deploy to production-like environment**
- **Document for handoff**

**Duration**: 2 weeks | **Weight**: 30% of final grade

### Certification (Optional)

Upon completion:
1. **Portfolio review**: Submit 5 projects for review
2. **Capstone presentation**: 30-min technical presentation
3. **Peer code review**: Review 2 peer projects
4. **Final assessment**: 4-hour practical exam

**Certificate**: ML Platform Engineer - Verified Competency

## Support & Community

### Getting Help

- **Documentation**: Check project READMEs and CURRICULUM.md first
- **Community**: See [community/](./community/) for FAQ-style notes (formal FAQ planned)
- **GitHub Issues**: [Open an issue](https://github.com/ai-infra-curriculum/ai-infra-ml-platform-learning/issues)
- **Discussions**: [GitHub Discussions](https://github.com/ai-infra-curriculum/ai-infra-ml-platform-learning/discussions)

### Community Channels

- **Primary channels**: [GitHub Discussions](https://github.com/ai-infra-curriculum/ai-infra-ml-platform-learning/discussions) and Issues are the supported community surfaces today.
- **Dedicated chat / office hours**: Not currently scheduled. Open a Discussion if you'd like to coordinate a study group or office-hours slot with other learners.

### Stay Updated

- **Star this repo** for updates
- **Watch releases** for new content
- **GitHub Discussions** is the active channel for announcements and updates.

## Contributing

We welcome contributions from the community! See [CONTRIBUTING.md](./CONTRIBUTING.md) for:

- Code of conduct
- How to submit issues
- Pull request process
- Development setup
- Testing guidelines
- Documentation standards

**Areas for contribution**:
- New exercises and labs
- Additional project ideas
- Documentation improvements
- Bug fixes and corrections
- Translation to other languages
- Cloud-specific guides (AWS, GCP, Azure)

## License

This project is licensed under the MIT License - see the [LICENSE](./LICENSE) file for details.

You are free to:
- Use this curriculum for personal learning
- Use in corporate training programs
- Modify and adapt for your needs
- Teach courses using this material

**Attribution appreciated but not required.**

## Acknowledgments

This curriculum is built on the collective knowledge of the ML platform engineering community:

- **Open Source Projects**: Kubeflow, MLflow, Feast, Airflow, and many others
- **Industry Leaders**: Companies pioneering ML platforms (Uber, Netflix, Airbnb, LinkedIn)
- **Academic Research**: Papers on ML systems and platform engineering
- **Community Contributors**: Everyone who has contributed feedback and improvements

### Special Thanks

- Kubernetes community for container orchestration patterns
- CNCF projects for cloud-native best practices
- MLOps community for workflow patterns
- Platform engineering community for DX insights

---

**Ready to begin your ML Platform Engineering journey?**

Start with [CURRICULUM.md](./CURRICULUM.md) for the complete learning guide, then dive into [Module 001](./lessons/mod-001-platform-fundamentals/).

**Questions?** [Open an issue](https://github.com/ai-infra-curriculum/ai-infra-ml-platform-learning/issues).

**Contact**: [ai-infra-curriculum@joshua-ferguson.com](mailto:ai-infra-curriculum@joshua-ferguson.com)

---

*Last updated: 2025-10-18 | Version: 1.0.0*


---

<!-- aicg:maintained-by -->
Maintained by [VeriSwarm.ai](https://veriswarm.ai)
