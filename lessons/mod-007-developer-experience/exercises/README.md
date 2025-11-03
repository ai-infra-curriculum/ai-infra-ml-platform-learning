# Module 07 Exercises: Developer Experience & SDKs

Build a complete developer experience layer for an ML platform with Python SDK, CLI, and documentation.

**Total Time**: ~8 hours
**Difficulty**: Intermediate to Advanced

---

## Exercise Overview

| Exercise | Title | Duration | Difficulty | Key Skills |
|----------|-------|----------|------------|------------|
| 01 | Build a Python SDK for Training Jobs | 90 min | Intermediate | SDK design, type hints, error handling |
| 02 | Create a CLI with Typer | 75 min | Intermediate | CLI patterns, argument parsing, output formatting |
| 03 | Implement SDK Authentication & Configuration | 60 min | Intermediate | Auth patterns, config management, environment variables |
| 04 | Build Self-Service Portal with Backstage | 120 min | Advanced | IDP, service catalog, templates |
| 05 | Create Jupyter Extension | 90 min | Advanced | Jupyter APIs, nbextensions, server extensions |
| 06 | Generate API Documentation with Sphinx | 45 min | Basic | Documentation, docstrings, Sphinx |

---

## Prerequisites

Before starting these exercises, ensure you have:

- [x] Completed Module 06 (Model Registry & Versioning)
- [x] Python 3.9+
- [x] Node.js 18+ (for Backstage)
- [x] Docker and Docker Compose
- [x] Basic understanding of REST APIs
- [x] Familiarity with CLI tools

**Installation**:
```bash
# Create virtual environment
python -m venv devexp-env
source devexp-env/bin/activate

# Install dependencies
pip install typer==0.9.0 \
    rich==13.7.0 \
    requests==2.31.0 \
    pydantic==2.5.0 \
    sphinx==7.2.0 \
    jupyter==1.0.0

# For Backstage (Exercise 04)
npm install -g @backstage/create-app
```

---

## Exercise 01: Build a Python SDK for Training Jobs

**Duration**: 90 minutes
**Difficulty**: Intermediate

### Learning Objectives

- Design idiomatic Python APIs
- Implement type-safe interfaces with dataclasses
- Provide actionable error messages
- Follow progressive disclosure pattern

### Scenario

You're building a Python SDK for your ML platform that allows data scientists to submit training jobs, monitor status, and retrieve logs. The SDK should feel natural to Python developers and require minimal boilerplate.

### Implementation Guide

**Step 1: Define Data Models**

```python
# ml_platform_sdk/models.py
from dataclasses import dataclass, field
from typing import Optional, Dict, List
from enum import Enum

class JobStatus(str, Enum):
    """Training job status."""
    PENDING = "PENDING"
    RUNNING = "RUNNING"
    SUCCEEDED = "SUCCEEDED"
    FAILED = "FAILED"
    CANCELLED = "CANCELLED"

@dataclass
class Resources:
    """Resource requirements for training jobs.

    Attributes:
        cpu: Number of CPU cores (default: 4)
        memory: Memory in Gi (default: "16Gi")
        gpu: Number of GPUs (default: 0)
        gpu_type: GPU model, e.g. "v100", "a100" (optional)

    Example:
        >>> resources = Resources(gpu=4, gpu_type="a100")
        >>> print(resources)
        Resources(cpu=4, memory='16Gi', gpu=4, gpu_type='a100')
    """
    cpu: int = 4
    memory: str = "16Gi"
    gpu: int = 0
    gpu_type: Optional[str] = None

    def __post_init__(self):
        """Validate resource values."""
        if self.cpu < 1:
            raise ValueError("cpu must be >= 1")
        if self.gpu < 0:
            raise ValueError("gpu must be >= 0")
        if self.gpu > 0 and self.gpu_type is None:
            # Default GPU type if not specified
            self.gpu_type = "t4"

@dataclass
class JobMetrics:
    """Training job metrics."""
    loss: float
    accuracy: float
    epoch: int
    timestamp: str

@dataclass
class TrainingJobConfig:
    """Complete configuration for a training job."""
    name: str
    script: str
    resources: Resources = field(default_factory=Resources)
    environment: Dict[str, str] = field(default_factory=dict)
    hyperparameters: Dict[str, any] = field(default_factory=dict)
    timeout_seconds: Optional[int] = None
```

**Step 2: Implement Custom Exceptions**

```python
# ml_platform_sdk/exceptions.py

class MLPlatformError(Exception):
    """Base exception for all SDK errors."""
    pass

class ValidationError(MLPlatformError):
    """Raised when job configuration is invalid."""

    def __init__(self, field: str, message: str):
        self.field = field
        super().__init__(f"Validation error in '{field}': {message}")

class ResourceUnavailableError(MLPlatformError):
    """Raised when requested resources cannot be allocated."""

    def __init__(self, requested: Resources, available: Resources):
        self.requested = requested
        self.available = available

        message = (
            f"Insufficient resources to schedule job.\n"
            f"Requested: {requested.gpu} GPUs ({requested.gpu_type})\n"
            f"Available: {available.gpu} GPUs\n\n"
            f"Suggestions:\n"
            f"1. Reduce GPU count: resources=Resources(gpu={available.gpu})\n"
            f"2. Wait for resources: job.submit(wait_for_capacity=True)\n"
            f"3. Use different GPU type (available types: t4, v100)"
        )
        super().__init__(message)

class JobNotFoundError(MLPlatformError):
    """Raised when job ID does not exist."""

    def __init__(self, job_id: str):
        super().__init__(
            f"Job '{job_id}' not found. "
            f"Verify the job ID or check if the job was deleted."
        )

class AuthenticationError(MLPlatformError):
    """Raised when authentication fails."""

    def __init__(self):
        super().__init__(
            "Authentication failed. Please check your API key.\n"
            "Set API key via:\n"
            "1. Environment variable: export ML_PLATFORM_API_KEY=your-key\n"
            "2. Config file: ~/.mlplatform/config.yaml\n"
            "3. Client initialization: MLPlatformClient(api_key='your-key')"
        )
```

**Step 3: Implement TrainingJob Class**

```python
# ml_platform_sdk/training.py
import time
from typing import Optional, List
from .models import JobStatus, Resources, JobMetrics, TrainingJobConfig
from .exceptions import JobNotFoundError, ResourceUnavailableError

class TrainingJob:
    """Distributed training job on the ML platform.

    This class provides an interface to submit, monitor, and manage
    training jobs. Jobs are executed on Kubernetes with requested resources.

    Attributes:
        name: Unique job identifier
        script: Path to training script
        resources: Compute resources for the job
        environment: Environment variables as key-value pairs
        id: Job ID (assigned after submission)
        status: Current job status

    Example:
        Create and submit a simple training job:

        >>> job = TrainingJob(
        ...     name="sentiment-analysis",
        ...     script="train.py",
        ...     resources=Resources(gpu=2)
        ... )
        >>> job.submit()
        >>> print(f"Job {job.id} submitted")
        >>> job.wait_for_completion()
        >>> print(f"Final status: {job.status}")
    """

    def __init__(
        self,
        name: str,
        script: str,
        resources: Optional[Resources] = None,
        environment: Optional[dict] = None,
        hyperparameters: Optional[dict] = None,
        client = None
    ):
        self.name = name
        self.script = script
        self.resources = resources or Resources()
        self.environment = environment or {}
        self.hyperparameters = hyperparameters or {}
        self._client = client

        # Set after submission
        self.id: Optional[str] = None
        self.status: Optional[JobStatus] = None
        self._logs_cache: List[str] = []

    def submit(self, wait_for_capacity: bool = False) -> str:
        """Submit job to the platform.

        Args:
            wait_for_capacity: If True, wait for resources to become available

        Returns:
            Job ID assigned by the platform

        Raises:
            ValidationError: Invalid job configuration
            ResourceUnavailableError: Insufficient resources

        Example:
            >>> job = TrainingJob(name="test", script="train.py")
            >>> job_id = job.submit()
            >>> print(f"Submitted: {job_id}")
        """
        # Validate configuration
        self._validate()

        # Submit to API
        response = self._client._post("/api/v1/jobs", {
            "name": self.name,
            "script": self.script,
            "resources": {
                "cpu": self.resources.cpu,
                "memory": self.resources.memory,
                "gpu": self.resources.gpu,
                "gpu_type": self.resources.gpu_type
            },
            "environment": self.environment,
            "hyperparameters": self.hyperparameters
        })

        self.id = response["job_id"]
        self.status = JobStatus(response["status"])

        return self.id

    def get_status(self) -> JobStatus:
        """Get current job status.

        Returns:
            Current job status

        Raises:
            JobNotFoundError: Job does not exist
        """
        if not self.id:
            raise ValueError("Job has not been submitted yet")

        response = self._client._get(f"/api/v1/jobs/{self.id}")
        self.status = JobStatus(response["status"])
        return self.status

    def get_logs(self, tail: int = 100, follow: bool = False) -> List[str]:
        """Get job logs.

        Args:
            tail: Number of lines to return from end of logs
            follow: If True, continuously stream new logs

        Returns:
            List of log lines

        Example:
            >>> job.get_logs(tail=50)
            ['Epoch 1/10', 'Loss: 0.45', ...]

            >>> for log in job.get_logs(follow=True):
            ...     print(log)
        """
        if not self.id:
            raise ValueError("Job has not been submitted yet")

        if follow:
            return self._stream_logs()

        response = self._client._get(
            f"/api/v1/jobs/{self.id}/logs",
            params={"tail": tail}
        )
        return response["logs"]

    def get_metrics(self) -> Optional[JobMetrics]:
        """Get training metrics.

        Returns:
            Latest training metrics, or None if not available
        """
        if not self.id:
            raise ValueError("Job has not been submitted yet")

        response = self._client._get(f"/api/v1/jobs/{self.id}/metrics")

        if not response["metrics"]:
            return None

        return JobMetrics(**response["metrics"])

    def wait_for_completion(
        self,
        timeout_seconds: Optional[int] = None,
        poll_interval: int = 5
    ) -> JobStatus:
        """Wait for job to complete.

        Args:
            timeout_seconds: Maximum time to wait (None = wait indefinitely)
            poll_interval: Seconds between status checks

        Returns:
            Final job status

        Raises:
            TimeoutError: Job did not complete within timeout
        """
        start_time = time.time()

        while True:
            status = self.get_status()

            if status in [JobStatus.SUCCEEDED, JobStatus.FAILED, JobStatus.CANCELLED]:
                return status

            if timeout_seconds and (time.time() - start_time) > timeout_seconds:
                raise TimeoutError(
                    f"Job did not complete within {timeout_seconds} seconds"
                )

            time.sleep(poll_interval)

    def cancel(self) -> None:
        """Cancel running job."""
        if not self.id:
            raise ValueError("Job has not been submitted yet")

        self._client._delete(f"/api/v1/jobs/{self.id}")
        self.status = JobStatus.CANCELLED

    def _validate(self):
        """Validate job configuration."""
        if not self.name:
            raise ValidationError("name", "Name cannot be empty")

        if not self.script:
            raise ValidationError("script", "Script path cannot be empty")

        # Add more validation as needed

    def _stream_logs(self):
        """Generator that yields log lines as they become available."""
        # Implementation for streaming logs
        pass

    def __repr__(self) -> str:
        return (
            f"TrainingJob(name='{self.name}', "
            f"id='{self.id}', "
            f"status={self.status})"
        )
```

**Step 4: Implement Client**

```python
# ml_platform_sdk/client.py
import os
import requests
from typing import Optional, List
from .training import TrainingJob
from .models import Resources
from .exceptions import AuthenticationError, MLPlatformError

class MLPlatformClient:
    """Client for ML Platform API.

    Example:
        >>> client = MLPlatformClient(api_key="your-key")
        >>> job = client.create_job(name="test", script="train.py")
        >>> job.submit()
    """

    def __init__(
        self,
        api_key: Optional[str] = None,
        base_url: Optional[str] = None,
        api_version: str = "v1"
    ):
        self.api_key = api_key or os.environ.get("ML_PLATFORM_API_KEY")
        if not self.api_key:
            raise AuthenticationError()

        self.base_url = base_url or os.environ.get(
            "ML_PLATFORM_URL",
            "https://api.mlplatform.example.com"
        )
        self.api_version = api_version

        self._session = requests.Session()
        self._session.headers.update({
            "Authorization": f"Bearer {self.api_key}",
            "Content-Type": "application/json"
        })

    def create_job(
        self,
        name: str,
        script: str,
        resources: Optional[Resources] = None,
        **kwargs
    ) -> TrainingJob:
        """Create a new training job.

        Args:
            name: Job name
            script: Path to training script
            resources: Resource requirements
            **kwargs: Additional job configuration

        Returns:
            TrainingJob instance
        """
        return TrainingJob(
            name=name,
            script=script,
            resources=resources,
            client=self,
            **kwargs
        )

    def get_job(self, job_id: str) -> TrainingJob:
        """Get existing job by ID.

        Args:
            job_id: Job identifier

        Returns:
            TrainingJob instance
        """
        response = self._get(f"/api/{self.api_version}/jobs/{job_id}")

        job = TrainingJob(
            name=response["name"],
            script=response["script"],
            resources=Resources(**response["resources"]),
            client=self
        )
        job.id = job_id
        job.status = response["status"]

        return job

    def list_jobs(
        self,
        status: Optional[str] = None,
        limit: int = 20
    ) -> List[TrainingJob]:
        """List training jobs.

        Args:
            status: Filter by status
            limit: Maximum number of jobs to return

        Returns:
            List of TrainingJob instances
        """
        params = {"limit": limit}
        if status:
            params["status"] = status

        response = self._get(f"/api/{self.api_version}/jobs", params=params)

        jobs = []
        for job_data in response["jobs"]:
            job = TrainingJob(
                name=job_data["name"],
                script=job_data["script"],
                resources=Resources(**job_data["resources"]),
                client=self
            )
            job.id = job_data["id"]
            job.status = job_data["status"]
            jobs.append(job)

        return jobs

    def _get(self, path: str, params: Optional[dict] = None) -> dict:
        """Make GET request."""
        url = f"{self.base_url}{path}"
        response = self._session.get(url, params=params)
        self._handle_response(response)
        return response.json()

    def _post(self, path: str, data: dict) -> dict:
        """Make POST request."""
        url = f"{self.base_url}{path}"
        response = self._session.post(url, json=data)
        self._handle_response(response)
        return response.json()

    def _delete(self, path: str) -> None:
        """Make DELETE request."""
        url = f"{self.base_url}{path}"
        response = self._session.delete(url)
        self._handle_response(response)

    def _handle_response(self, response: requests.Response):
        """Handle API response errors."""
        if response.status_code == 401:
            raise AuthenticationError()
        elif response.status_code == 404:
            raise JobNotFoundError(response.json().get("job_id", "unknown"))
        elif response.status_code >= 400:
            raise MLPlatformError(f"API error: {response.text}")
```

### Success Criteria

- [ ] SDK allows creating and submitting training jobs
- [ ] Type hints work correctly with IDE autocomplete
- [ ] Custom exceptions provide actionable error messages
- [ ] Jobs can be monitored for status and logs
- [ ] Client handles authentication properly
- [ ] Code passes type checking with mypy

### Testing

```python
# test_sdk.py
from ml_platform_sdk import MLPlatformClient, Resources

def test_job_submission():
    client = MLPlatformClient(api_key="test-key")

    job = client.create_job(
        name="test-job",
        script="train.py",
        resources=Resources(gpu=2, gpu_type="v100")
    )

    assert job.name == "test-job"
    assert job.resources.gpu == 2

    # Mock submission
    # job.submit()
    # assert job.id is not None
    # assert job.status == JobStatus.PENDING

if __name__ == "__main__":
    test_job_submission()
    print("✓ All tests passed")
```

---

## Exercise 02: Create a CLI with Typer

**Duration**: 75 minutes
**Difficulty**: Intermediate

### Learning Objectives

- Build intuitive CLIs following UNIX conventions
- Support multiple output formats (table, JSON, YAML)
- Implement interactive prompts
- Use colors and formatting for better UX

### Implementation

```python
# ml_platform_cli/main.py
import typer
from typing import Optional
from enum import Enum
from rich.console import Console
from rich.table import Table
from ml_platform_sdk import MLPlatformClient, Resources

app = typer.Typer(
    name="mlplatform",
    help="ML Platform CLI for managing training jobs"
)
console = Console()

class OutputFormat(str, Enum):
    TABLE = "table"
    JSON = "json"
    YAML = "yaml"

@app.command()
def init():
    """Initialize ML Platform configuration."""
    api_key = typer.prompt("Enter your API key", hide_input=True)
    base_url = typer.prompt(
        "Enter platform URL",
        default="https://api.mlplatform.example.com"
    )

    # Save to config file
    config_dir = Path.home() / ".mlplatform"
    config_dir.mkdir(exist_ok=True)

    config_file = config_dir / "config.yaml"
    config_file.write_text(f"""api_key: {api_key}
base_url: {base_url}
""")

    typer.secho("✓ Configuration saved", fg=typer.colors.GREEN)

@app.command("list")
def list_jobs(
    status: Optional[str] = typer.Option(None, "--status", "-s"),
    output: OutputFormat = typer.Option(OutputFormat.TABLE, "--output", "-o"),
    limit: int = typer.Option(20, "--limit", "-l")
):
    """List training jobs."""
    client = MLPlatformClient()
    jobs = client.list_jobs(status=status, limit=limit)

    if output == OutputFormat.TABLE:
        table = Table(title="Training Jobs")
        table.add_column("Name", style="cyan")
        table.add_column("ID", style="magenta")
        table.add_column("Status", style="yellow")
        table.add_column("GPU", style="green")
        table.add_column("Created")

        for job in jobs:
            table.add_row(
                job.name,
                job.id,
                job.status.value,
                str(job.resources.gpu),
                "2h ago"  # Would come from API
            )

        console.print(table)

    elif output == OutputFormat.JSON:
        import json
        data = [
            {
                "name": job.name,
                "id": job.id,
                "status": job.status.value,
                "gpu": job.resources.gpu
            }
            for job in jobs
        ]
        print(json.dumps(data, indent=2))

    else:  # YAML
        import yaml
        data = [
            {
                "name": job.name,
                "id": job.id,
                "status": job.status.value,
                "gpu": job.resources.gpu
            }
            for job in jobs
        ]
        print(yaml.dump(data))

@app.command()
def create(
    name: str = typer.Argument(..., help="Job name"),
    script: str = typer.Option(..., "--script", help="Training script path"),
    gpu: int = typer.Option(1, "--gpu", help="Number of GPUs"),
    gpu_type: str = typer.Option("t4", "--gpu-type", help="GPU type"),
    memory: str = typer.Option("16Gi", "--memory", help="Memory"),
    interactive: bool = typer.Option(False, "--interactive", "-i"),
    submit: bool = typer.Option(True, "--submit/--no-submit")
):
    """Create a new training job."""

    if interactive:
        name = typer.prompt("Job name", default=name)
        script = typer.prompt("Training script", default=script)

        gpu = typer.prompt(
            "GPU count",
            type=int,
            default=gpu,
            show_choices=False
        )

        gpu_type = typer.prompt(
            "GPU type",
            default=gpu_type,
            type=typer.Choice(["t4", "v100", "a100"])
        )

    client = MLPlatformClient()
    resources = Resources(gpu=gpu, gpu_type=gpu_type, memory=memory)

    job = client.create_job(name=name, script=script, resources=resources)

    if submit:
        with console.status("[bold green]Submitting job..."):
            job.submit()

        typer.secho(f"✓ Job created: {job.id}", fg=typer.colors.GREEN)
        typer.echo(f"  Name: {job.name}")
        typer.echo(f"  GPU: {job.resources.gpu} x {job.resources.gpu_type}")
        typer.echo(f"  Status: {job.status.value}")
    else:
        typer.echo(f"Job configuration created (not submitted)")

@app.command()
def logs(
    job_id: str = typer.Argument(..., help="Job ID"),
    follow: bool = typer.Option(False, "--follow", "-f"),
    tail: int = typer.Option(100, "--tail", "-n")
):
    """Get job logs."""
    client = MLPlatformClient()
    job = client.get_job(job_id)

    logs = job.get_logs(tail=tail, follow=follow)

    for log in logs:
        typer.echo(log)

@app.command()
def status(job_id: str = typer.Argument(..., help="Job ID")):
    """Get job status."""
    client = MLPlatformClient()
    job = client.get_job(job_id)

    status = job.get_status()

    # Color based on status
    color = {
        "PENDING": typer.colors.YELLOW,
        "RUNNING": typer.colors.BLUE,
        "SUCCEEDED": typer.colors.GREEN,
        "FAILED": typer.colors.RED,
        "CANCELLED": typer.colors.MAGENTA
    }.get(status.value, typer.colors.WHITE)

    typer.secho(f"Job {job_id}: {status.value}", fg=color)

    # Show metrics if available
    metrics = job.get_metrics()
    if metrics:
        typer.echo(f"  Epoch: {metrics.epoch}")
        typer.echo(f"  Loss: {metrics.loss:.4f}")
        typer.echo(f"  Accuracy: {metrics.accuracy:.4f}")

@app.command()
def cancel(job_id: str = typer.Argument(..., help="Job ID")):
    """Cancel a running job."""
    confirm = typer.confirm(f"Cancel job {job_id}?")

    if not confirm:
        typer.echo("Cancelled")
        return

    client = MLPlatformClient()
    job = client.get_job(job_id)
    job.cancel()

    typer.secho(f"✓ Job {job_id} cancelled", fg=typer.colors.YELLOW)

if __name__ == "__main__":
    app()
```

### Success Criteria

- [ ] CLI follows standard UNIX conventions
- [ ] Commands support table, JSON, and YAML output
- [ ] Interactive mode works for job creation
- [ ] Colors and formatting enhance readability
- [ ] Help text is clear and comprehensive

### Testing

```bash
# Initialize
mlplatform init

# List jobs
mlplatform list
mlplatform list --status RUNNING --output json

# Create job
mlplatform create my-job --script train.py --gpu 4 --gpu-type v100
mlplatform create --interactive

# Get logs
mlplatform logs job-abc123
mlplatform logs job-abc123 --follow

# Check status
mlplatform status job-abc123

# Cancel job
mlplatform cancel job-abc123
```

---

## Exercise 03: Implement SDK Authentication & Configuration

**Duration**: 60 minutes
**Difficulty**: Intermediate

### Learning Objectives

- Implement multiple authentication methods
- Load configuration from files and environment
- Handle API keys securely
- Provide clear error messages for auth issues

### Implementation

```python
# ml_platform_sdk/config.py
import os
from pathlib import Path
from typing import Optional
import yaml

class Config:
    """Configuration management for ML Platform SDK."""

    DEFAULT_CONFIG_PATH = Path.home() / ".mlplatform" / "config.yaml"

    def __init__(
        self,
        api_key: Optional[str] = None,
        base_url: Optional[str] = None,
        config_file: Optional[Path] = None
    ):
        self.config_file = config_file or self.DEFAULT_CONFIG_PATH
        self._load_config()

        # Priority: explicit params > env vars > config file
        self.api_key = (
            api_key or
            os.environ.get("ML_PLATFORM_API_KEY") or
            self._config.get("api_key")
        )

        self.base_url = (
            base_url or
            os.environ.get("ML_PLATFORM_URL") or
            self._config.get("base_url") or
            "https://api.mlplatform.example.com"
        )

        self.timeout = int(
            os.environ.get("ML_PLATFORM_TIMEOUT") or
            self._config.get("timeout") or
            30
        )

    def _load_config(self):
        """Load configuration from file."""
        if self.config_file.exists():
            with open(self.config_file) as f:
                self._config = yaml.safe_load(f) or {}
        else:
            self._config = {}

    def save(self):
        """Save current configuration to file."""
        self.config_file.parent.mkdir(parents=True, exist_ok=True)

        with open(self.config_file, 'w') as f:
            yaml.dump({
                "api_key": self.api_key,
                "base_url": self.base_url,
                "timeout": self.timeout
            }, f)

    @classmethod
    def from_file(cls, path: Path) -> "Config":
        """Load configuration from specific file."""
        return cls(config_file=path)
```

### Success Criteria

- [ ] Config loads from file, environment, or parameters
- [ ] Priority order is respected (params > env > file)
- [ ] API keys are never logged or printed
- [ ] Config can be saved to file
- [ ] Clear error messages when config is missing

---

## Exercise 04: Build Self-Service Portal with Backstage

**Duration**: 120 minutes
**Difficulty**: Advanced

### Implementation

```bash
# Create Backstage app
npx @backstage/create-app --skip-install

cd my-backstage-app
yarn install
yarn dev
```

**Custom Plugin**:
```typescript
// plugins/ml-platform/src/components/JobsPage.tsx
import React from 'react';
import { Page, Header, Content } from '@backstage/core-components';
import { JobsTable } from './JobsTable';

export const JobsPage = () => {
  return (
    <Page themeId="tool">
      <Header title="Training Jobs" subtitle="Manage ML training jobs" />
      <Content>
        <JobsTable />
      </Content>
    </Page>
  );
};
```

### Success Criteria

- [ ] Backstage running locally
- [ ] Service catalog populated with ML services
- [ ] Self-service template for job creation
- [ ] Custom plugin displays training jobs

---

## Exercise 05: Create Jupyter Extension

**Duration**: 90 minutes
**Difficulty**: Advanced

### Implementation

```python
# jupyter_mlplatform/__init__.py
def _jupyter_server_extension_paths():
    return [{
        "module": "jupyter_mlplatform"
    }]

def load_jupyter_server_extension(nbapp):
    from .handlers import setup_handlers
    setup_handlers(nbapp.web_app)
```

### Success Criteria

- [ ] Extension adds "Submit Job" button to toolbar
- [ ] Jobs can be submitted from notebook
- [ ] Status updates shown in notebook

---

## Exercise 06: Generate API Documentation with Sphinx

**Duration**: 45 minutes
**Difficulty**: Basic

### Implementation

```bash
# Generate docs
sphinx-quickstart docs
sphinx-apidoc -o docs/source ml_platform_sdk
sphinx-build docs/source docs/build
```

### Success Criteria

- [ ] API documentation auto-generated
- [ ] Examples included in docstrings
- [ ] Documentation builds without errors
- [ ] HTML output looks professional

---

## Additional Resources

- [Typer Documentation](https://typer.tiangolo.com/)
- [Backstage Documentation](https://backstage.io/docs/)
- [Jupyter Extension Guide](https://jupyter-notebook.readthedocs.io/en/stable/extending/)
- [Sphinx Documentation](https://www.sphinx-doc.org/)

---

**Status**: ✅ Complete | **Last Updated**: November 2, 2025
