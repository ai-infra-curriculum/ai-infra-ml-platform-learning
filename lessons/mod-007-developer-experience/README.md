# Module 07: Developer Experience & SDKs

Build developer-friendly tools that make your ML platform a joy to use. Learn SDK design, CLI development, and internal developer portals.

**Duration**: 7-8 hours (lecture + exercises)
**Difficulty**: Intermediate to Advanced
**Prerequisites**: Modules 01-06 completed

---

## Learning Objectives

By the end of this module, you will be able to:

- [ ] Design idiomatic Python SDKs following best practices
- [ ] Build command-line interfaces with Typer
- [ ] Implement authentication and configuration management
- [ ] Create self-service portals with Backstage
- [ ] Build Jupyter extensions for seamless integration
- [ ] Generate professional API documentation
- [ ] Measure and improve developer experience metrics

---

## Why Developer Experience Matters

**The Problem**: Even powerful ML platforms fail if they're hard to use.
- Data scientists bypass your platform for familiar tools
- Support teams spend 60% of time answering "how do I...?" questions
- Onboarding new team members takes weeks instead of days
- Inconsistent usage patterns create maintenance nightmares

**The Solution**: Great DX accelerates adoption and reduces support burden.
- **Uber**: Reduced model deployment time from 3 weeks to 3 days with self-service tools
- **Airbnb**: Increased platform adoption from 30% to 90% after DX improvements
- **Netflix**: Open-sourced Metaflow due to internal success - data scientists build production pipelines in days

---

## Module Structure

### 📚 Lecture Notes (90 minutes)

#### [01: Developer Experience & SDKs](./lecture-notes/01-developer-experience-sdks.md)

**Topics Covered**:

1. **Why DX Matters**
   - Low adoption without great UX
   - Real-world impact (Uber, Airbnb, Netflix)
   - Key principles: self-service, discoverability, consistency

2. **Python SDK Design Principles**
   - Idiomatic Python (not REST wrappers)
   - Progressive disclosure (simple simple, complex possible)
   - Type safety with type hints
   - Sensible defaults
   - Actionable error messages

3. **CLI Design Patterns**
   - UNIX conventions
   - Multiple output formats (table, JSON, YAML)
   - Interactive mode with prompts
   - Implementation with Typer

4. **Internal Developer Portals**
   - Service catalogs
   - Self-service actions
   - Backstage architecture
   - Custom plugin development

5. **Jupyter Extensions**
   - Notebook extensions
   - Server extensions
   - JupyterLab extensions
   - Platform integration

6. **Documentation Best Practices**
   - Four types: tutorials, guides, reference, explanation
   - Auto-generated API docs with Sphinx
   - Code examples in docstrings

7. **SDK Versioning**
   - Semantic versioning
   - Deprecation strategies
   - API version negotiation

8. **Measuring DX**
   - Time to first success
   - API call success rate
   - Support ticket volume
   - NPS scores

**Key Code Example** (SDK Design):
```python
from ml_platform import MLPlatformClient, Resources

client = MLPlatformClient()

# Simple (90% use case)
job = client.create_job(name="training", script="train.py")
job.submit()

# Advanced (full control)
job = client.create_job(
    name="training",
    script="train.py",
    resources=Resources(gpu=4, gpu_type="a100", memory="64Gi"),
    environment={"NCCL_DEBUG": "INFO"},
    retry_policy=RetryPolicy(max_attempts=3)
)
job.submit()
```

**Key Code Example** (CLI with Typer):
```python
import typer
from rich.console import Console

app = typer.Typer()
console = Console()

@app.command()
def create(
    name: str,
    script: str = typer.Option(..., "--script"),
    gpu: int = typer.Option(1, "--gpu"),
    interactive: bool = typer.Option(False, "-i")
):
    """Create a new training job."""
    if interactive:
        name = typer.prompt("Job name", default=name)
        gpu = typer.prompt("GPU count", type=int, default=gpu)

    client = MLPlatformClient()
    job = client.create_job(name=name, script=script, resources=Resources(gpu=gpu))
    job.submit()

    typer.secho(f"✓ Job created: {job.id}", fg=typer.colors.GREEN)
```

---

### 🛠️ Hands-On Exercises (8 hours)

Complete 6 comprehensive exercises building a full developer experience layer:

#### Exercise 01: Build a Python SDK for Training Jobs (90 min, Intermediate)
- Design idiomatic Python APIs with dataclasses
- Implement type-safe interfaces
- Create actionable error messages
- Build TrainingJob and MLPlatformClient classes

**Success Criteria**: SDK allows creating/submitting jobs with full type safety

---

#### Exercise 02: Create a CLI with Typer (75 min, Intermediate)
- Follow UNIX conventions (list, create, logs, status, cancel)
- Support multiple output formats (table, JSON, YAML)
- Implement interactive prompts
- Use Rich for beautiful terminal output

**Success Criteria**: Full-featured CLI with colors, tables, and progress bars

---

#### Exercise 03: SDK Authentication & Configuration (60 min, Intermediate)
- Multiple auth methods (API key, env vars, config file)
- Priority order: params > env > file
- Secure handling of credentials
- YAML configuration management

**Success Criteria**: Config loads correctly with proper priority order

---

#### Exercise 04: Self-Service Portal with Backstage (120 min, Advanced)
- Set up Backstage with custom plugin
- Define service catalog for ML platform
- Create self-service templates for jobs
- Build custom React components

**Success Criteria**: Running Backstage portal with ML platform integration

---

#### Exercise 05: Jupyter Extension (90 min, Advanced)
- Build server extension with REST endpoints
- Add toolbar button to notebooks
- Submit jobs directly from Jupyter
- Display status updates in UI

**Success Criteria**: "Submit Job" button works in Jupyter notebooks

---

#### Exercise 06: Generate API Documentation (45 min, Basic)
- Write comprehensive docstrings with examples
- Auto-generate docs with Sphinx
- Build HTML documentation
- Host docs locally

**Success Criteria**: Professional-looking HTML docs with examples

---

## Tools & Technologies

**Required**:
- Python 3.9+
- Typer 0.9+ (CLI framework)
- Rich 13.7+ (terminal formatting)
- Requests 2.31+ (HTTP client)
- Pydantic 2.5+ (data validation)
- Sphinx 7.2+ (documentation)
- Node.js 18+ (for Backstage)
- Docker & Docker Compose

**Python Packages**:
```bash
pip install typer==0.9.0 \
    rich==13.7.0 \
    requests==2.31.0 \
    pydantic==2.5.0 \
    sphinx==7.2.0 \
    jupyter==1.0.0 \
    pyyaml==6.0
```

**For Backstage**:
```bash
npm install -g @backstage/create-app
```

---

## Prerequisites

Before starting this module, ensure you have:

- [x] Completed Module 06 (Model Registry & Versioning)
- [x] Strong Python programming skills
- [x] Familiarity with REST APIs
- [x] Basic understanding of CLI tools
- [x] React basics (for Backstage plugin)

**Recommended Background**:
- Experience building Python libraries
- Familiarity with type hints and dataclasses
- Understanding of HTTP authentication
- Basic frontend development (HTML/CSS/JS)

---

## Time Breakdown

| Component | Duration | Format |
|-----------|----------|--------|
| Lecture notes | 90 min | Reading + code review |
| Exercise 01 (SDK) | 90 min | Hands-on coding |
| Exercise 02 (CLI) | 75 min | Hands-on coding |
| Exercise 03 (Config) | 60 min | Hands-on coding |
| Exercise 04 (Backstage) | 120 min | Hands-on setup + coding |
| Exercise 05 (Jupyter) | 90 min | Hands-on coding |
| Exercise 06 (Docs) | 45 min | Documentation |
| **Total** | **~9.5 hours** | Mixed |

**Recommended Schedule**:
- **Day 1**: Lecture + Exercise 01-02 (4 hours)
- **Day 2**: Exercise 03-04 (3 hours)
- **Day 3**: Exercise 05-06 (2.5 hours)

---

## Success Criteria

You have successfully completed this module when you can:

1. **SDK Design** ✅
   - Build idiomatic Python APIs
   - Implement type-safe interfaces with dataclasses
   - Provide actionable error messages
   - Follow progressive disclosure pattern

2. **CLI Development** ✅
   - Create intuitive CLIs following UNIX conventions
   - Support multiple output formats
   - Implement interactive prompts with Typer
   - Use Rich for beautiful terminal output

3. **Configuration Management** ✅
   - Load config from files, environment, and parameters
   - Respect priority order (params > env > file)
   - Handle API keys securely
   - Provide clear error messages for missing config

4. **Internal Developer Portal** ✅
   - Set up Backstage locally
   - Create service catalog entries
   - Build self-service templates
   - Develop custom React plugins

5. **Jupyter Integration** ✅
   - Build Jupyter server extensions
   - Add UI elements to notebooks
   - Submit jobs from Jupyter
   - Display status updates

6. **Documentation** ✅
   - Write comprehensive docstrings
   - Generate API docs with Sphinx
   - Include code examples
   - Build professional HTML output

---

## Real-World Applications

### Uber: Michelangelo SDK
- **Challenge**: 200+ ML use cases, diverse teams, manual deployment taking weeks
- **Solution**: Python SDK + CLI with self-service capabilities
- **Impact**: Model deployment time reduced from 3 weeks to 3 days
- **Key Features**: Simple API for 90% use cases, advanced options for power users

### Airbnb: Bighead Platform
- **Challenge**: Only 30% of ML projects using platform due to complexity
- **Solution**: Rebuilt SDK with idiomatic Python, added Jupyter integration
- **Impact**: Platform adoption increased to 90%, onboarding time cut in half
- **Key Features**: Progressive disclosure, sensible defaults, excellent error messages

### Netflix: Metaflow
- **Challenge**: Data scientists spending months building production infrastructure
- **Solution**: Open-sourced Python framework with decorator-based API
- **Impact**: Production pipelines built in days instead of months
- **Key Features**: Pythonic API, automatic versioning, built-in retry logic

---

## Common Pitfalls

### 1. Over-Abstracting
**Problem**: SDK tries to hide too much, limiting power users

**Solution**: Follow progressive disclosure - simple simple, complex possible
```python
# Good: Allow both simple and advanced usage
job = Job(name="test", script="train.py")  # Simple
job = Job(name="test", script="train.py", retry_policy=...)  # Advanced
```

### 2. Poor Error Messages
**Problem**: Generic errors like "Job failed" without context

**Solution**: Provide actionable errors with remediation suggestions
```python
raise ResourceUnavailableError(
    f"Insufficient resources.\n"
    f"Requested: 4 GPUs (a100)\n"
    f"Available: 2 GPUs\n\n"
    f"Suggestions:\n"
    f"1. Reduce GPU count: Resources(gpu=2)\n"
    f"2. Wait for capacity: job.submit(wait_for_capacity=True)"
)
```

### 3. Breaking Changes
**Problem**: Frequent API changes frustrate users

**Solution**: Use semantic versioning and deprecation warnings
```python
warnings.warn(
    "Parameter 'num_gpus' deprecated. Use 'resources=Resources(gpu=X)'",
    DeprecationWarning
)
```

### 4. Missing Type Hints
**Problem**: No IDE autocomplete, errors caught at runtime

**Solution**: Use type hints everywhere
```python
def create_job(
    name: str,
    script: str,
    resources: Optional[Resources] = None
) -> TrainingJob:
    ...
```

### 5. Inadequate Documentation
**Problem**: Users can't figure out how to use features

**Solution**: Four types of docs (tutorials, guides, reference, explanation)

---

## Assessment

### Knowledge Check (after lecture notes)

1. What are the five principles of good SDK design?
2. Why is progressive disclosure important?
3. What's the difference between a tutorial and a how-to guide?
4. How does Backstage improve developer experience?
5. What's the priority order for configuration loading?

### Practical Assessment (after exercises)

Build a complete developer experience layer that:
- [ ] Provides Python SDK for training jobs
- [ ] Includes CLI with list, create, logs, status, cancel commands
- [ ] Supports multiple auth methods (API key, env, config file)
- [ ] Generates professional API documentation
- [ ] Integrates with Jupyter notebooks
- [ ] Follows all best practices from lecture

**Acceptance Criteria**:
- SDK has full type hints and passes mypy
- CLI supports table/JSON/YAML output
- Config loads with correct priority order
- Documentation includes examples in docstrings
- Jupyter extension adds toolbar button successfully
- Error messages are actionable and helpful

---

## Additional Resources

### Official Documentation
- [Typer Documentation](https://typer.tiangolo.com/)
- [Rich Documentation](https://rich.readthedocs.io/)
- [Backstage Documentation](https://backstage.io/docs/)
- [Sphinx Documentation](https://www.sphinx-doc.org/)
- [Jupyter Extension Guide](https://jupyter-notebook.readthedocs.io/en/stable/extending/)

### Tutorials & Guides
- [Building Great SDKs](https://newsletter.pragmaticengineer.com/p/building-great-sdks) - Pragmatic Engineer
- [Python SDK Design Patterns](https://liblab.com/docs/languages/python)
- [Platform Engineering Patterns](https://platformengineering.org/platform-tooling)
- [CLI Guidelines](https://clig.dev/)

### Real-World Examples
- [Netflix Metaflow](https://metaflow.org/)
- [Uber Michelangelo](https://www.uber.com/blog/michelangelo-model-representation/)
- [Airbnb Bighead](https://medium.com/airbnb-engineering/ml-platform-at-airbnb-79fcb9e1b037)

### Video Tutorials
- [Building Python CLIs with Typer](https://www.youtube.com/results?search_query=typer+python+cli)
- [Backstage Introduction](https://www.youtube.com/results?search_query=backstage+tutorial)

---

## Troubleshooting

### SDK Installation Issues

**Symptom**: `ImportError: cannot import name 'MLPlatformClient'`

**Solutions**:
```bash
# Ensure package is installed in editable mode
pip install -e .

# Verify installation
python -c "from ml_platform_sdk import MLPlatformClient; print('OK')"

# Check Python path
python -c "import sys; print(sys.path)"
```

---

### CLI Not Found

**Symptom**: `mlplatform: command not found`

**Solutions**:
```bash
# Install CLI in editable mode
pip install -e .[cli]

# Or use python -m
python -m ml_platform_cli --help

# Add to setup.py entry_points
[console_scripts]
mlplatform=ml_platform_cli.main:app
```

---

### Backstage Build Errors

**Symptom**: `ERROR in ./src/index.tsx`

**Solutions**:
```bash
# Clear cache and reinstall
rm -rf node_modules yarn.lock
yarn install

# Check Node version (requires 18+)
node --version

# Rebuild
yarn tsc:full
yarn build
```

---

### Type Checking Fails

**Symptom**: mypy errors about missing type hints

**Solutions**:
```python
# Add type: ignore for third-party libraries
from third_party import Foo  # type: ignore

# Create stub file (.pyi)
# ml_platform_sdk/client.pyi
from typing import Optional
class MLPlatformClient:
    def __init__(self, api_key: Optional[str] = None) -> None: ...
```

---

## Next Steps

After completing this module, you're ready for:

- **Module 08: Observability & Monitoring** - Monitor models and infrastructure
- **Module 09: Security & Governance** - Implement RBAC, audit logs, compliance

**Recommended Path**: Proceed to Module 08 to learn how to instrument your ML platform with metrics, logs, and traces for production observability.

---

## Feedback & Support

**Questions?** Open an issue in the repository with the `module-07` tag.

**Found a bug in the code?** Submit a PR with the fix.

**Want more exercises?** Check the `/exercises/bonus/` directory for advanced challenges.

---

**Status**: ✅ Complete | **Last Updated**: November 2, 2025 | **Version**: 1.0.0
