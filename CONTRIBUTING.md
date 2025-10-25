# Contributing to ML Platform Engineer Learning Repository

Thank you for your interest in contributing to the AI Infrastructure Curriculum! This repository helps experienced engineers specialize in ML Platform Engineering. Your contributions help make this advanced learning resource better for everyone.

## Table of Contents

- [Code of Conduct](#code-of-conduct)
- [How Can I Contribute?](#how-can-i-contribute)
- [Getting Started](#getting-started)
- [Contribution Guidelines](#contribution-guidelines)
- [Pull Request Process](#pull-request-process)
- [Style Guides](#style-guides)
- [Community](#community)

## Code of Conduct

This project and everyone participating in it is governed by our [Code of Conduct](CODE_OF_CONDUCT.md). By participating, you are expected to uphold this code. Please report unacceptable behavior to ai-infra-curriculum@joshua-ferguson.com.

## How Can I Contribute?

### Reporting Bugs or Issues

If you find a bug in the code, documentation, or learning materials:

1. **Check existing issues** to see if it's already been reported
2. If not, **create a new issue** using our bug report template
3. Include:
   - Clear, descriptive title
   - Steps to reproduce the issue
   - Expected behavior vs. actual behavior
   - Your environment (OS, Python version, Kubernetes version, etc.)
   - Screenshots or error messages if applicable
   - Relevant platform components (API, feature store, workflow engine, etc.)

### Suggesting Enhancements

Have an idea to improve the platform engineering curriculum?

1. **Check existing issues** to avoid duplicates
2. **Create a new issue** using our feature request template
3. Clearly describe:
   - The problem or gap you've identified
   - Your proposed solution
   - Why this would benefit learners
   - Any alternative solutions you've considered
   - Platform engineering best practices that support your proposal

### Improving Documentation

Documentation improvements are always welcome! This includes:

- Fixing typos or grammatical errors
- Clarifying complex architectural concepts
- Adding architecture diagrams
- Improving code comments
- Expanding the README or guides
- Adding real-world platform examples
- Documenting platform design patterns

For small fixes (typos, grammar), feel free to submit a PR directly. For larger changes, please open an issue first to discuss.

### Contributing Code or Exercises

We welcome contributions of:

- Bug fixes to existing code stubs
- Improvements to platform architecture examples
- Additional test cases for platform components
- Better API design examples
- New resources or references
- Kubernetes operator examples
- Feature store implementation improvements

**Important**: This is a **learning repository** for advanced engineers. Code stubs should remain incomplete with clear TODOs that guide learners through platform implementation. Do not submit complete solutions to this repository (those belong in the solutions repository).

### Improving Learning Materials

Help make platform engineering lessons better by:

- Clarifying distributed systems concepts
- Adding real-world platform architecture examples
- Creating system design diagrams
- Suggesting additional resources (Backstage, Kubeflow, Feast documentation)
- Improving exercise instructions for complex topics
- Adding practice problems for API design
- Documenting platform engineering patterns

## Getting Started

### Prerequisites

- Python 3.11 or higher
- Docker and Kubernetes (minikube or kind for local development)
- Git and GitHub account
- Experience with infrastructure engineering
- Understanding of distributed systems (recommended)

### Setting Up Your Development Environment

1. **Fork the repository** on GitHub

2. **Clone your fork** locally:
   ```bash
   git clone https://github.com/YOUR-USERNAME/ai-infra-ml-platform-learning.git
   cd ai-infra-ml-platform-learning
   ```

3. **Add the upstream repository**:
   ```bash
   git remote add upstream https://github.com/ai-infra-curriculum/ai-infra-ml-platform-learning.git
   ```

4. **Create a virtual environment**:
   ```bash
   python -m venv venv
   source venv/bin/activate  # On Windows: venv\Scripts\activate
   ```

5. **Install dependencies**:
   ```bash
   pip install -r requirements.txt
   pip install -e .  # Install in development mode
   ```

6. **Install pre-commit hooks** (optional but recommended):
   ```bash
   pre-commit install
   ```

7. **Set up local Kubernetes** (for platform development):
   ```bash
   # Using minikube
   minikube start --cpus=4 --memory=8192

   # Or using kind
   kind create cluster --config=dev/kind-config.yaml
   ```

### Running Tests

Before submitting changes, ensure all tests pass:

```bash
# Run all tests
pytest

# Run with coverage
pytest --cov

# Run specific test file
pytest tests/test_platform_api.py

# Test Kubernetes integration
pytest tests/integration/test_k8s_controller.py
```

### Code Quality Checks

We use several tools to maintain code quality:

```bash
# Format code with black
black .

# Sort imports
isort .

# Lint with flake8
flake8 .

# Type checking with mypy
mypy .

# Or run all checks at once with pre-commit
pre-commit run --all-files
```

## Contribution Guidelines

### General Principles

1. **Keep it Educational**: Remember this is an advanced learning resource for platform engineering. Code should demonstrate best practices and include architectural explanations.

2. **Maintain Difficulty Level**: This is the ML Platform Engineer repository. Content should be appropriate for experienced engineers building production platforms.

3. **Follow Existing Patterns**: Match the style and structure of existing platform components.

4. **Test Your Changes**: Ensure code works and all platform integration tests pass.

5. **Document Everything**: Platform engineering requires comprehensive documentation. Explain architectural decisions and trade-offs.

6. **Consider Scale**: Platform designs should work for hundreds of users and teams.

### What Makes a Good Contribution?

Good contributions are:

- **Clear and focused**: One logical change per PR
- **Well-tested**: Include or update tests, especially integration tests
- **Well-documented**: Explain what changed, why, and architectural implications
- **Pedagogically sound**: Helps learners understand platform engineering concepts
- **Production-ready**: Demonstrates real-world platform patterns
- **Scalable**: Shows how to design for multi-tenant, high-scale platforms
- **Complete**: Includes all necessary files, tests, and documentation

### What We're NOT Looking For

Please avoid:

- Complete solutions to exercises (use the solutions repo instead)
- Basic topics covered in junior/senior engineer curricula
- Large refactors without prior discussion
- Changes that break existing platform integrations
- Poorly documented or tested code
- Plagiarized content
- Non-scalable platform patterns

## Pull Request Process

### Before You Submit

1. **Create a new branch** for your changes:
   ```bash
   git checkout -b feature/platform-api-enhancement
   # or
   git checkout -b fix/feature-store-bug
   ```

2. **Keep your fork updated**:
   ```bash
   git fetch upstream
   git rebase upstream/main
   ```

3. **Make your changes** following our guidelines

4. **Run tests and quality checks**:
   ```bash
   pytest
   black .
   flake8 .
   mypy .
   # Test platform components
   pytest tests/integration/
   ```

5. **Commit your changes** with clear messages:
   ```bash
   git add .
   git commit -m "feat: add multi-tenant resource quota controller"
   ```

   We follow [Conventional Commits](https://www.conventionalcommits.org/):
   - `feat:` new features or platform capabilities
   - `fix:` bug fixes
   - `docs:` documentation changes
   - `test:` adding or updating tests
   - `refactor:` code refactoring
   - `style:` formatting changes
   - `chore:` maintenance tasks

6. **Push to your fork**:
   ```bash
   git push origin feature/platform-api-enhancement
   ```

### Submitting Your PR

1. Go to the original repository on GitHub
2. Click "New Pull Request"
3. Select your fork and branch
4. Fill out the PR template completely
5. Link any related issues
6. Describe architectural decisions if applicable
7. Submit the PR

### PR Review Process

1. **Automated checks** will run (tests, linting, integration tests)
2. **Maintainers will review** your code (usually within 3-5 days)
3. **Address feedback** by pushing new commits to your branch
4. **Architecture review** for platform design changes
5. Once approved, a maintainer will **merge your PR**

### After Your PR is Merged

1. **Delete your branch**:
   ```bash
   git branch -d feature/platform-api-enhancement
   git push origin --delete feature/platform-api-enhancement
   ```

2. **Update your fork**:
   ```bash
   git checkout main
   git pull upstream main
   git push origin main
   ```

3. **Celebrate!** You've contributed to platform engineering education!

## Style Guides

### Python Code Style

- Follow [PEP 8](https://pep8.org/)
- Use [Black](https://black.readthedocs.io/) for formatting (line length: 88)
- Use [isort](https://pycqa.github.io/isort/) for import sorting
- Add type hints using Python 3.11+ syntax
- Write docstrings in [Google style](https://google.github.io/styleguide/pyguide.html#38-comments-and-docstrings)

Example for platform components:

```python
from typing import List, Optional, Dict, Any
from pydantic import BaseModel


class PlatformResourceRequest(BaseModel):
    """Request model for platform resource provisioning.
    
    This model defines the schema for requesting ML infrastructure
    resources through the platform API. It supports multi-tenancy
    and resource quotas.
    
    Attributes:
        team_id: Unique identifier for the team/namespace
        resource_type: Type of resource (jupyter, training_job, deployment)
        cpu: CPU cores requested (e.g., "4" or "8")
        memory: Memory requested (e.g., "16Gi" or "32Gi")
        gpu: Number of GPUs (0 for CPU-only)
        labels: Additional metadata for resource tracking
    """
    
    team_id: str
    resource_type: str
    cpu: str = "2"
    memory: str = "4Gi"
    gpu: int = 0
    labels: Optional[Dict[str, str]] = None
    
    def validate_against_quota(self, quota: 'ResourceQuota') -> bool:
        """Validate this request against team quota.
        
        Args:
            quota: The team's resource quota object
            
        Returns:
            True if request is within quota, False otherwise
            
        Raises:
            ValueError: If quota validation logic encounters an error
        """
        # TODO: Implement quota validation logic
        pass
```

### Documentation Style

- Use clear, technical language appropriate for experienced engineers
- Define platform engineering concepts when first used
- Include architecture diagrams for complex systems
- Document API contracts with OpenAPI specs
- Provide examples of platform usage patterns
- Explain multi-tenancy and scaling considerations
- Use headers to organize content logically

### Markdown Style

- Use ATX-style headers (`#` not underlines)
- One blank line before and after headers
- Use fenced code blocks with language specification
- Keep line length reasonable (80-100 characters)
- Use relative links for internal references
- Include Mermaid diagrams for architecture

### Commit Message Style

Follow [Conventional Commits](https://www.conventionalcommits.org/):

```
type(scope): brief description

Longer explanation if needed. Wrap at 72 characters.

- Bullet points are okay
- Use present tense: "add" not "added"
- Reference issues: "fixes #123"
- Explain architectural decisions

BREAKING CHANGE: description of breaking change if applicable
```

Types:
- `feat`: New platform feature or capability
- `fix`: Bug fix
- `docs`: Documentation changes
- `test`: Test changes
- `refactor`: Code refactoring
- `perf`: Performance improvements
- `chore`: Maintenance

## Community

### Getting Help

- **Issues**: Browse existing issues or create a new one
- **Discussions**: Use GitHub Discussions for platform design questions
- **Email**: ai-infra-curriculum@joshua-ferguson.com

### Recognition

Contributors will be:
- Listed in our Contributors section
- Credited in release notes
- Mentioned in our community highlights
- Recognized for significant platform contributions

### Staying Connected

- Watch the repository for updates
- Star the repo to show support
- Share your platform engineering journey with `#AIInfraCurriculum #MLPlatform`
- Join platform engineering communities (CNCF, MLOps Community)

## Questions?

Don't hesitate to ask about platform architecture or design decisions:

- Open a GitHub Discussion for architectural questions
- Comment on relevant issues
- Email us at ai-infra-curriculum@joshua-ferguson.com

Thank you for helping make ML Platform Engineering education better for everyone!

---

**Remember**: This is a specialized learning community for experienced engineers. We value deep technical discussion, thoughtful architecture, and helping others build production platforms. Every contribution, no matter how small, makes a difference.
