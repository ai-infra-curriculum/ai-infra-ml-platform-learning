# Exercise 04: Build a Simple Plugin System

## Objective

Design a plugin interface for *training backends* in an ML platform, then walk through an implementation in Python. You will write the abstract interface as Python Protocol classes, implement two backends (a local mock backend for testing, and a Kubernetes-flavored backend), write a registry/loader that selects the right backend at runtime, and document the operational concerns of running plugins in production.

You will write real Python code in this exercise. The code is intentionally illustrative — designed to be readable end-to-end in one sitting — but it should run if you copy-paste it into a Python ≥ 3.10 environment.

## Learning Outcomes

By completing this exercise, you will:

- Design a Python Protocol that serves as a plugin interface.
- Implement two concrete plugins against the interface.
- Build a registry pattern for discovering and selecting plugins.
- Recognize the operational risks of in-process plugins and articulate mitigations.
- Compare in-process plugins to subprocess and webhook alternatives.

## Prerequisites

- Read Lecture 05 (Platform Architecture Patterns), especially the "Plugin Systems and Extension Points" section.
- Comfortable reading and writing Python (no advanced features needed beyond type hints + classes + protocols).
- Python 3.10+ available on your machine, or a willingness to read code without running it.

## Scenario

Continuing the Aurelia AI platform scenario: your training-jobs API needs to dispatch submitted training jobs to *backends* that actually run them. Today you have one backend (Kubernetes), but the team is starting to talk about:

- A *local* backend for development — runs the training in a subprocess on the developer's machine.
- An *AWS Batch* backend for one team that already has Batch infrastructure.
- A *Vertex AI* backend for a research team that prefers Google's tooling.
- A *mock* backend for testing the platform itself.

Rather than write four similar implementations, you decide to design a *training backend* plugin interface that any of these can implement. This exercise builds that interface.

## Deliverables

By the end of this exercise, you will have created:

1. `plugin/interface.py` — the Protocol classes defining the plugin contract.
2. `plugin/registry.py` — the registry/loader.
3. `plugin/backends/mock.py` — the mock backend (used in tests).
4. `plugin/backends/k8s.py` — a Kubernetes-flavored backend (without real Kubernetes client; illustrative).
5. `plugin/example_usage.py` — a tiny script that exercises both backends through the registry.
6. `tests/test_plugin.py` — at least 4 tests against the mock backend.
7. `plugin/README.md` — operational documentation for the plugin system.

All files live under a `plugin/` directory you create as part of this exercise.

---

## Part 1: Plugin Interface (25 minutes)

### Task 1.1: Domain model

Before writing the protocol, define the data types the plugin will work with. Create `plugin/types.py`:

```python
"""Domain types shared across the plugin interface."""

from __future__ import annotations

from dataclasses import dataclass, field
from datetime import datetime
from enum import Enum
from typing import Any


class JobStatus(str, Enum):
    """Lifecycle states for a training job. Matches the public API enum."""

    PENDING = "pending"
    SCHEDULING = "scheduling"
    RUNNING = "running"
    SUCCEEDED = "succeeded"
    FAILED = "failed"
    CANCELLED = "cancelled"


@dataclass(frozen=True)
class ResourceSpec:
    """Resources requested by a training job."""

    cpus: float = 1.0           # cores
    memory_gi: float = 4.0      # gibibytes
    gpus: int = 0
    gpu_type: str | None = None  # e.g., "a100", "h100"
    ephemeral_storage_gi: float = 10.0


@dataclass(frozen=True)
class TrainingJobSpec:
    """A platform-internal training job specification.

    This is what the platform's API translates user requests into.
    Plugins should operate on this shape, not on raw HTTP request bodies.
    """

    job_id: str                   # opaque ID assigned by the platform
    name: str                     # human-readable name
    tenant: str                   # owning tenant
    owner: str                    # submitting user
    code_image: str               # container image with the training code
    code_command: list[str]       # command to execute
    code_args: list[str] = field(default_factory=list)
    env: dict[str, str] = field(default_factory=dict)
    resources: ResourceSpec = field(default_factory=ResourceSpec)
    artifact_output_uri: str | None = None  # e.g., s3://...
    dataset_uris: list[str] = field(default_factory=list)
    metadata: dict[str, str] = field(default_factory=dict)


@dataclass(frozen=True)
class JobHandle:
    """An opaque reference to a submitted job, returned by a backend.

    Different backends have different native IDs (k8s uid, AWS Batch ARN,
    etc.). The JobHandle wraps those in a uniform shape.
    """

    backend: str            # name of the backend, e.g., "kubernetes"
    native_id: str          # the backend-specific ID
    submitted_at: datetime


@dataclass(frozen=True)
class JobStatusReport:
    """Latest known status of a job."""

    handle: JobHandle
    status: JobStatus
    started_at: datetime | None
    completed_at: datetime | None
    exit_code: int | None
    last_message: str | None    # last status message (e.g., "node out of memory")
    backend_details: dict[str, Any] = field(default_factory=dict)


class BackendUnavailableError(Exception):
    """Raised when a backend cannot be reached at all."""


class JobNotFoundError(Exception):
    """Raised when a job handle refers to something the backend doesn't know."""


class InvalidJobSpecError(Exception):
    """Raised when the backend cannot accept the given TrainingJobSpec."""
```

**TODO**: Create `plugin/types.py` with this content (or your own equivalent). Note the design choices:

- `ResourceSpec` is `frozen=True` — immutable, hashable.
- `TrainingJobSpec` is the *platform-internal* shape; the API translates user requests into this.
- `JobHandle` is opaque to the platform; only the backend can dereference it.
- Errors are custom exceptions so the platform can catch them specifically.

Reflect: why is `gpu_type` `str | None` rather than an enum? (Hint: forward compatibility — see Lecture 04's discussion of enum forward compatibility.)

### Task 1.2: The Protocol class

Create `plugin/interface.py`:

```python
"""Protocol definitions for training-backend plugins."""

from __future__ import annotations

from typing import Iterator, Protocol, runtime_checkable

from .types import (
    JobHandle,
    JobStatusReport,
    TrainingJobSpec,
)


@runtime_checkable
class TrainingBackend(Protocol):
    """A pluggable backend that can run training jobs.

    A backend is responsible for:
        1. Accepting a TrainingJobSpec and returning a JobHandle for tracking.
        2. Reporting current status when asked.
        3. Cancelling a running job.
        4. Streaming logs.

    A backend is NOT responsible for:
        - Validating the user's permission to submit the job.
        - Allocating quota or enforcing fair-sharing.
        - Translating user-facing API errors into HTTP responses.

    Those are the platform's responsibilities. The backend's job is to
    take a well-formed TrainingJobSpec and run it; everything else is
    handled upstream.

    Implementations must be safe to call from multiple threads.
    """

    #: Stable name used in the registry, e.g., "kubernetes", "mock".
    name: str

    #: Human-readable description (shown in admin tooling).
    description: str

    def submit(self, spec: TrainingJobSpec) -> JobHandle:
        """Submit a training job.

        Raises:
            InvalidJobSpecError: if the backend cannot run this spec
                (e.g., requested 100 GPUs when the backend has 8).
            BackendUnavailableError: if the backend cannot be reached.
        """
        ...

    def status(self, handle: JobHandle) -> JobStatusReport:
        """Query the current status of a previously-submitted job.

        Raises:
            JobNotFoundError: if the handle refers to nothing the
                backend knows about.
            BackendUnavailableError: if the backend cannot be reached.
        """
        ...

    def cancel(self, handle: JobHandle) -> None:
        """Cancel a running job.

        Cancelling a non-running job (already succeeded, failed, etc.)
        is a no-op and does not raise.

        Raises:
            JobNotFoundError: if the handle refers to nothing the
                backend knows about.
            BackendUnavailableError: if the backend cannot be reached.
        """
        ...

    def logs(self, handle: JobHandle, follow: bool = False) -> Iterator[str]:
        """Stream stdout/stderr from the job.

        If follow=False, returns all logs to date and ends the iterator.
        If follow=True, continues to yield new log lines until the job
        terminates or the iterator is closed.

        Raises:
            JobNotFoundError: if the handle refers to nothing the
                backend knows about.
        """
        ...
```

**TODO**: Create `plugin/interface.py`. Walk through the docstrings; note the explicit "is not responsible for" list. That's the contract discipline that prevents plugins from sprawling.

Reflect: the protocol uses `@runtime_checkable` — what does that buy you? (Hint: it lets you `isinstance(obj, TrainingBackend)`, which is occasionally useful for validating loaded plugins; without it, the Protocol is structural-only.)

### Task 1.3: Reflect on what's *not* in the interface

A common failure mode is to keep adding methods to the interface ("we need a `get_metrics()` method, and a `set_priority()` method, and a `get_quota_usage()` method ...").

**TODO**: In `plugin/README.md`, write a section "What the Interface Deliberately Excludes" listing at least 4 things that *could* be on the interface but aren't, and why.

Examples to start you off:

- **Quota/usage queries.** The platform tracks quota, not the backend.
- **Pricing.** The platform's cost model is platform-side.
- **Identity.** The platform passes opaque tokens; the backend doesn't see user identity.
- **Notification/email.** The platform notifies users; the backend just runs jobs.

---

## Part 2: Mock Backend (15 minutes)

### Task 2.1: Implement the mock

Create `plugin/backends/__init__.py` (empty) and `plugin/backends/mock.py`:

```python
"""In-memory mock backend for testing.

This backend does not actually execute anything. It simulates a job's
lifecycle by transitioning through states based on elapsed time, which
makes it useful for testing the platform's higher-level logic.
"""

from __future__ import annotations

import time
import threading
import uuid
from datetime import datetime, timezone
from typing import Iterator

from ..interface import TrainingBackend  # noqa: F401  (just to assert conformance)
from ..types import (
    BackendUnavailableError,
    InvalidJobSpecError,
    JobHandle,
    JobNotFoundError,
    JobStatus,
    JobStatusReport,
    TrainingJobSpec,
)


class MockBackend:
    """A test-only backend that simulates job execution.

    Configurable:
        run_duration_s: how long a "running" job stays running before
            it transitions to succeeded.
        always_fail: if True, jobs transition to failed instead of
            succeeded.
        unavailable: if True, every call raises BackendUnavailableError.
    """

    name = "mock"
    description = "In-memory mock for testing. Does not actually run code."

    def __init__(
        self,
        *,
        run_duration_s: float = 1.0,
        always_fail: bool = False,
        unavailable: bool = False,
    ) -> None:
        self._jobs: dict[str, dict] = {}  # native_id -> internal state
        self._lock = threading.Lock()
        self._run_duration_s = run_duration_s
        self._always_fail = always_fail
        self._unavailable = unavailable

    # --- TrainingBackend protocol methods ---

    def submit(self, spec: TrainingJobSpec) -> JobHandle:
        if self._unavailable:
            raise BackendUnavailableError("mock backend is in unavailable mode")
        if spec.resources.gpus > 100:
            # arbitrary limit so we can exercise the InvalidJobSpec path
            raise InvalidJobSpecError(
                f"mock backend refuses jobs with >100 GPUs (got {spec.resources.gpus})"
            )
        native_id = f"mock-{uuid.uuid4().hex[:12]}"
        now = datetime.now(timezone.utc)
        with self._lock:
            self._jobs[native_id] = {
                "spec": spec,
                "submitted_at": now,
                "started_at": now,           # mock has no real scheduling delay
                "status": JobStatus.RUNNING,
                "logs": [
                    f"[{now.isoformat()}] mock backend starting job {spec.name}",
                ],
            }
        return JobHandle(backend="mock", native_id=native_id, submitted_at=now)

    def status(self, handle: JobHandle) -> JobStatusReport:
        if self._unavailable:
            raise BackendUnavailableError("mock backend is in unavailable mode")
        with self._lock:
            job = self._jobs.get(handle.native_id)
            if job is None:
                raise JobNotFoundError(handle.native_id)
            # Maybe transition based on elapsed time
            self._maybe_transition(handle.native_id)
            job = self._jobs[handle.native_id]
            return JobStatusReport(
                handle=handle,
                status=job["status"],
                started_at=job.get("started_at"),
                completed_at=job.get("completed_at"),
                exit_code=job.get("exit_code"),
                last_message=job.get("last_message"),
                backend_details={"mock_internal_id": handle.native_id},
            )

    def cancel(self, handle: JobHandle) -> None:
        if self._unavailable:
            raise BackendUnavailableError("mock backend is in unavailable mode")
        with self._lock:
            job = self._jobs.get(handle.native_id)
            if job is None:
                raise JobNotFoundError(handle.native_id)
            if job["status"] in (
                JobStatus.SUCCEEDED, JobStatus.FAILED, JobStatus.CANCELLED
            ):
                # idempotent: cancelling a finished job is a no-op
                return
            job["status"] = JobStatus.CANCELLED
            job["completed_at"] = datetime.now(timezone.utc)
            job["last_message"] = "cancelled by request"
            job["logs"].append(
                f"[{job['completed_at'].isoformat()}] cancelled by request"
            )

    def logs(self, handle: JobHandle, follow: bool = False) -> Iterator[str]:
        if self._unavailable:
            raise BackendUnavailableError("mock backend is in unavailable mode")
        # Snapshot existing logs
        with self._lock:
            job = self._jobs.get(handle.native_id)
            if job is None:
                raise JobNotFoundError(handle.native_id)
            snapshot = list(job["logs"])
        yield from snapshot

        if not follow:
            return

        # Follow mode: poll for new log lines until job ends
        last_len = len(snapshot)
        while True:
            time.sleep(0.05)
            with self._lock:
                job = self._jobs.get(handle.native_id)
                if job is None:
                    raise JobNotFoundError(handle.native_id)
                current_logs = job["logs"]
                status = job["status"]
            if len(current_logs) > last_len:
                for line in current_logs[last_len:]:
                    yield line
                last_len = len(current_logs)
            if status in (JobStatus.SUCCEEDED, JobStatus.FAILED, JobStatus.CANCELLED):
                return

    # --- internal ---

    def _maybe_transition(self, native_id: str) -> None:
        """Lazily move running jobs through their lifecycle."""
        job = self._jobs.get(native_id)
        if job is None:
            return
        if job["status"] != JobStatus.RUNNING:
            return
        elapsed = (
            datetime.now(timezone.utc) - job["started_at"]
        ).total_seconds()
        if elapsed < self._run_duration_s:
            return
        # Time to terminate
        end_status = JobStatus.FAILED if self._always_fail else JobStatus.SUCCEEDED
        exit_code = 1 if self._always_fail else 0
        completed_at = datetime.now(timezone.utc)
        job["status"] = end_status
        job["completed_at"] = completed_at
        job["exit_code"] = exit_code
        job["last_message"] = "simulated completion"
        job["logs"].append(
            f"[{completed_at.isoformat()}] mock job finished: {end_status.value}"
        )
```

**TODO**: Create `plugin/backends/mock.py` with this content. Notice:

- The class doesn't inherit from `TrainingBackend`. It conforms *structurally* — Python's Protocol does duck typing.
- The internal state is locked because the protocol promises thread-safety.
- The `unavailable` flag is for tests that exercise the platform's error-handling path.

### Task 2.2: Confirm protocol conformance

In `plugin/example_usage.py`, write:

```python
from plugin.backends.mock import MockBackend
from plugin.interface import TrainingBackend

backend = MockBackend()
assert isinstance(backend, TrainingBackend), "MockBackend must conform to TrainingBackend"
print(f"MockBackend conforms; name={backend.name}")
```

Running this should print without an assertion error.

**TODO**: Add this conformance check to `plugin/example_usage.py`.

---

## Part 3: Kubernetes-flavored Backend (20 minutes)

This backend would, in real life, call the Kubernetes API to submit a `Job`. We won't actually call Kubernetes here (the goal is to *understand the shape*, not run a cluster), but we'll write code that looks plausible.

### Task 3.1: Sketch

Create `plugin/backends/k8s.py`:

```python
"""Kubernetes-flavored training backend.

This backend translates a TrainingJobSpec into a Kubernetes Job manifest,
submits it via the Kubernetes API, and tracks its status by polling.

In a real implementation, we would use the `kubernetes` Python client.
For the purposes of this exercise, we stub those calls behind a protocol
so the code is illustrative without requiring a live cluster.
"""

from __future__ import annotations

import logging
import uuid
from datetime import datetime, timezone
from typing import Any, Iterator, Protocol

from ..types import (
    BackendUnavailableError,
    InvalidJobSpecError,
    JobHandle,
    JobNotFoundError,
    JobStatus,
    JobStatusReport,
    TrainingJobSpec,
)

log = logging.getLogger(__name__)


class KubernetesAPI(Protocol):
    """The narrow slice of Kubernetes API we depend on. Real impl wraps
    the official `kubernetes` python client; tests can substitute a fake."""

    def create_namespaced_job(self, namespace: str, body: dict) -> dict: ...
    def read_namespaced_job(self, name: str, namespace: str) -> dict: ...
    def delete_namespaced_job(self, name: str, namespace: str) -> None: ...
    def read_namespaced_job_log(
        self, name: str, namespace: str, follow: bool = False
    ) -> Iterator[str]: ...


class KubernetesBackend:
    """Submits training jobs as Kubernetes Jobs.

    Each job lands in the tenant's namespace, runs as the workload
    ServiceAccount, and inherits the tenant's quotas and network
    policies (see Exercise 02).
    """

    name = "kubernetes"
    description = "Runs training jobs as Kubernetes Jobs in the tenant namespace."

    def __init__(
        self,
        k8s_api: KubernetesAPI,
        *,
        namespace_prefix: str = "tenant-",
        service_account: str = "workload",
        default_image_pull_policy: str = "IfNotPresent",
    ) -> None:
        self._api = k8s_api
        self._namespace_prefix = namespace_prefix
        self._service_account = service_account
        self._default_image_pull_policy = default_image_pull_policy
        # Map JobHandle.native_id (which is "<namespace>/<name>") back to spec
        self._submitted: dict[str, TrainingJobSpec] = {}

    def submit(self, spec: TrainingJobSpec) -> JobHandle:
        self._validate_spec(spec)
        namespace = self._namespace_for(spec.tenant)
        k8s_name = self._make_k8s_name(spec.name)
        manifest = self._build_manifest(spec, namespace, k8s_name)
        try:
            self._api.create_namespaced_job(namespace=namespace, body=manifest)
        except Exception as exc:
            log.exception("k8s create_namespaced_job failed")
            raise BackendUnavailableError(str(exc)) from exc
        native_id = f"{namespace}/{k8s_name}"
        self._submitted[native_id] = spec
        return JobHandle(
            backend="kubernetes",
            native_id=native_id,
            submitted_at=datetime.now(timezone.utc),
        )

    def status(self, handle: JobHandle) -> JobStatusReport:
        namespace, name = self._parse_native_id(handle.native_id)
        try:
            job = self._api.read_namespaced_job(name=name, namespace=namespace)
        except Exception as exc:
            if "not found" in str(exc).lower():
                raise JobNotFoundError(handle.native_id) from exc
            raise BackendUnavailableError(str(exc)) from exc
        status, started_at, completed_at, exit_code, msg = (
            self._interpret_job_status(job)
        )
        return JobStatusReport(
            handle=handle,
            status=status,
            started_at=started_at,
            completed_at=completed_at,
            exit_code=exit_code,
            last_message=msg,
            backend_details={
                "k8s_namespace": namespace,
                "k8s_job_name": name,
                "k8s_conditions": job.get("status", {}).get("conditions"),
            },
        )

    def cancel(self, handle: JobHandle) -> None:
        namespace, name = self._parse_native_id(handle.native_id)
        try:
            self._api.delete_namespaced_job(name=name, namespace=namespace)
        except Exception as exc:
            if "not found" in str(exc).lower():
                # idempotent: cancel of non-existent job is a no-op
                return
            raise BackendUnavailableError(str(exc)) from exc

    def logs(self, handle: JobHandle, follow: bool = False) -> Iterator[str]:
        namespace, name = self._parse_native_id(handle.native_id)
        try:
            yield from self._api.read_namespaced_job_log(
                name=name, namespace=namespace, follow=follow
            )
        except Exception as exc:
            if "not found" in str(exc).lower():
                raise JobNotFoundError(handle.native_id) from exc
            raise BackendUnavailableError(str(exc)) from exc

    # --- internal helpers ---

    def _validate_spec(self, spec: TrainingJobSpec) -> None:
        if not spec.code_image:
            raise InvalidJobSpecError("code_image is required")
        if not spec.code_command:
            raise InvalidJobSpecError("code_command is required")
        if spec.resources.gpus < 0:
            raise InvalidJobSpecError("gpus must be >= 0")
        # Add more validation as needed (image registry allow-list, etc.)

    def _namespace_for(self, tenant: str) -> str:
        return f"{self._namespace_prefix}{tenant}"

    def _make_k8s_name(self, base: str) -> str:
        # K8s resource names: lowercase, RFC 1123 subdomain
        slug = "".join(c if c.isalnum() else "-" for c in base.lower()).strip("-")
        suffix = uuid.uuid4().hex[:6]
        return f"{slug[:50]}-{suffix}"

    def _parse_native_id(self, native_id: str) -> tuple[str, str]:
        try:
            namespace, name = native_id.split("/", 1)
        except ValueError as exc:
            raise JobNotFoundError(native_id) from exc
        return namespace, name

    def _build_manifest(
        self, spec: TrainingJobSpec, namespace: str, name: str
    ) -> dict[str, Any]:
        resources: dict[str, Any] = {
            "requests": {
                "cpu": str(spec.resources.cpus),
                "memory": f"{spec.resources.memory_gi}Gi",
                "ephemeral-storage": f"{spec.resources.ephemeral_storage_gi}Gi",
            },
            "limits": {
                "cpu": str(spec.resources.cpus * 2),
                "memory": f"{spec.resources.memory_gi * 2}Gi",
            },
        }
        if spec.resources.gpus > 0:
            resources["requests"]["nvidia.com/gpu"] = str(spec.resources.gpus)
            resources["limits"]["nvidia.com/gpu"] = str(spec.resources.gpus)
        env_list = [{"name": k, "value": v} for k, v in spec.env.items()]
        return {
            "apiVersion": "batch/v1",
            "kind": "Job",
            "metadata": {
                "name": name,
                "namespace": namespace,
                "labels": {
                    "platform.aurelia.example.com/tenant": spec.tenant,
                    "platform.aurelia.example.com/owner": spec.owner,
                    "platform.aurelia.example.com/job-id": spec.job_id,
                },
                "annotations": {
                    "platform.aurelia.example.com/job-name": spec.name,
                },
            },
            "spec": {
                "backoffLimit": 0,
                "template": {
                    "spec": {
                        "serviceAccountName": self._service_account,
                        "restartPolicy": "Never",
                        "containers": [
                            {
                                "name": "trainer",
                                "image": spec.code_image,
                                "imagePullPolicy": self._default_image_pull_policy,
                                "command": spec.code_command,
                                "args": spec.code_args,
                                "env": env_list,
                                "resources": resources,
                            }
                        ],
                    }
                },
            },
        }

    def _interpret_job_status(
        self, job: dict[str, Any]
    ) -> tuple[JobStatus, datetime | None, datetime | None, int | None, str | None]:
        status = job.get("status", {})
        if status.get("succeeded", 0) >= 1:
            return (
                JobStatus.SUCCEEDED,
                self._parse_ts(status.get("startTime")),
                self._parse_ts(status.get("completionTime")),
                0,
                "completed successfully",
            )
        if status.get("failed", 0) >= 1:
            return (
                JobStatus.FAILED,
                self._parse_ts(status.get("startTime")),
                self._parse_ts(status.get("completionTime")),
                1,
                "job failed",
            )
        if status.get("active", 0) >= 1:
            return (
                JobStatus.RUNNING,
                self._parse_ts(status.get("startTime")),
                None,
                None,
                "running",
            )
        return (JobStatus.PENDING, None, None, None, "pending scheduler")

    @staticmethod
    def _parse_ts(ts: str | None) -> datetime | None:
        if ts is None:
            return None
        # K8s timestamps are RFC 3339; datetime.fromisoformat works for most cases
        try:
            return datetime.fromisoformat(ts.replace("Z", "+00:00"))
        except ValueError:
            return None
```

**TODO**: Create `plugin/backends/k8s.py`. Read through the code; the goal is to internalize *what a backend implementation looks like* with a real-world target.

### Task 3.2: Reflect

In `plugin/README.md`, write 3-4 sentences each on:

1. **Why the `KubernetesAPI` protocol exists.** Why doesn't the backend just import `kubernetes` directly? (Hint: testability — you can substitute a fake.)
2. **The `_build_manifest` method's tenant-label injection.** What downstream systems depend on these labels?
3. **The `backoffLimit: 0` choice.** Why not let Kubernetes retry on failure? (Hint: the platform's user can retry from the platform side, with their full context; auto-retry inside Kubernetes is opaque.)

---

## Part 4: Registry (10 minutes)

### Task 4.1: Implement the registry

Create `plugin/registry.py`:

```python
"""Registry/loader for training backends."""

from __future__ import annotations

import logging
from typing import Iterable

from .interface import TrainingBackend

log = logging.getLogger(__name__)


class BackendRegistry:
    """Holds the set of available backends and selects between them.

    The platform builds one registry at startup. As backends are loaded
    (statically from config, or dynamically from plugin entry points),
    they are registered here. The platform then asks the registry to
    `pick()` a backend for each incoming job.
    """

    def __init__(self) -> None:
        self._backends: dict[str, TrainingBackend] = {}

    def register(self, backend: TrainingBackend) -> None:
        if not isinstance(backend, TrainingBackend):
            raise TypeError(
                f"backend {backend!r} does not conform to TrainingBackend"
            )
        name = backend.name
        if name in self._backends:
            log.warning("backend %s already registered; overwriting", name)
        self._backends[name] = backend
        log.info("registered backend: %s", name)

    def unregister(self, name: str) -> None:
        self._backends.pop(name, None)

    def names(self) -> list[str]:
        return sorted(self._backends.keys())

    def get(self, name: str) -> TrainingBackend:
        try:
            return self._backends[name]
        except KeyError as exc:
            raise KeyError(
                f"no backend named {name!r} (available: {self.names()})"
            ) from exc

    def pick(self, preferred: str | None = None) -> TrainingBackend:
        """Return a backend.

        If `preferred` is set, return that backend (or raise KeyError).
        If not set, pick a deterministic default (first by name).
        """
        if preferred is not None:
            return self.get(preferred)
        if not self._backends:
            raise RuntimeError("no backends registered")
        first = sorted(self._backends.keys())[0]
        return self._backends[first]


def discover_backends_from_entry_points() -> Iterable[TrainingBackend]:
    """Discover backends declared via setuptools entry points.

    Plugin authors declare entry points in their package's pyproject.toml:

        [project.entry-points."aurelia.training_backends"]
        my_backend = "my_pkg.module:MyBackend"

    We iterate them, instantiate, and yield. If instantiation fails for
    one, we log and skip — one bad plugin should not crash the platform.
    """
    # This uses importlib.metadata; available in Python 3.10+ stdlib.
    from importlib.metadata import entry_points
    eps = entry_points(group="aurelia.training_backends")
    for ep in eps:
        try:
            cls = ep.load()
            instance = cls()
            yield instance
        except Exception:
            log.exception("failed to load backend from entry point %r", ep.name)
            continue
```

**TODO**: Create `plugin/registry.py`. Trace through it; note:

- Backends are *instances*, not classes. The platform passes them around as objects.
- The registry can be populated either statically (in code, by calling `register()`) or dynamically (by scanning entry points).
- `discover_backends_from_entry_points` *isolates failures* — one bad plugin doesn't crash the platform. This is a non-negotiable property of any plugin loader.

### Task 4.2: Wire it up

Extend `plugin/example_usage.py`:

```python
import logging
from datetime import datetime, timezone
from unittest.mock import MagicMock

from plugin.backends.k8s import KubernetesBackend
from plugin.backends.mock import MockBackend
from plugin.interface import TrainingBackend
from plugin.registry import BackendRegistry
from plugin.types import ResourceSpec, TrainingJobSpec

logging.basicConfig(level=logging.INFO)


def main() -> None:
    # Build a registry with two backends
    registry = BackendRegistry()
    registry.register(MockBackend(run_duration_s=0.1))

    fake_k8s = MagicMock()
    fake_k8s.create_namespaced_job.return_value = {"status": "ok"}
    fake_k8s.read_namespaced_job.return_value = {"status": {"active": 1, "startTime": "2026-05-22T22:00:00Z"}}
    registry.register(KubernetesBackend(fake_k8s))

    # Inspect what's registered
    print(f"Backends registered: {registry.names()}")
    for name in registry.names():
        b = registry.get(name)
        assert isinstance(b, TrainingBackend)
        print(f"  - {b.name}: {b.description}")

    # Submit a job to each
    spec = TrainingJobSpec(
        job_id="job-0001",
        name="example-training",
        tenant="ml-research",
        owner="alice@aurelia.example",
        code_image="registry.aurelia.example.com/training:latest",
        code_command=["python", "train.py"],
        resources=ResourceSpec(cpus=4, memory_gi=16, gpus=1, gpu_type="a100"),
    )

    for name in registry.names():
        b = registry.pick(preferred=name)
        handle = b.submit(spec)
        print(f"[{name}] submitted: {handle.native_id}")
        status = b.status(handle)
        print(f"[{name}] status: {status.status.value}")


if __name__ == "__main__":
    main()
```

**TODO**: Save this. Run it (`python -m plugin.example_usage` from the parent directory) if you have Python available. Expected output (illustrative):

```
Backends registered: ['kubernetes', 'mock']
  - kubernetes: Runs training jobs as Kubernetes Jobs in the tenant namespace.
  - mock: In-memory mock for testing. Does not actually run code.
[kubernetes] submitted: tenant-ml-research/example-training-XXXXXX
[kubernetes] status: running
[mock] submitted: mock-XXXXXXXXXXXX
[mock] status: running
```

(If you cannot run Python here, walk through the code mentally to satisfy yourself it would work.)

---

## Part 5: Tests (10 minutes)

### Task 5.1: Write tests

Create `tests/__init__.py` (empty) and `tests/test_plugin.py`:

```python
"""Smoke tests for the plugin interface and mock backend."""

from __future__ import annotations

import time

import pytest

from plugin.backends.mock import MockBackend
from plugin.interface import TrainingBackend
from plugin.registry import BackendRegistry
from plugin.types import (
    BackendUnavailableError,
    InvalidJobSpecError,
    JobHandle,
    JobNotFoundError,
    JobStatus,
    ResourceSpec,
    TrainingJobSpec,
)


@pytest.fixture
def spec() -> TrainingJobSpec:
    return TrainingJobSpec(
        job_id="job-test",
        name="test-job",
        tenant="ml-research",
        owner="alice@aurelia.example",
        code_image="busybox:latest",
        code_command=["echo", "hello"],
        resources=ResourceSpec(cpus=1, memory_gi=2, gpus=0),
    )


def test_mock_backend_conforms_to_protocol() -> None:
    backend = MockBackend()
    assert isinstance(backend, TrainingBackend)
    assert backend.name == "mock"


def test_mock_backend_submit_returns_handle(spec: TrainingJobSpec) -> None:
    backend = MockBackend()
    handle = backend.submit(spec)
    assert handle.backend == "mock"
    assert handle.native_id.startswith("mock-")


def test_mock_backend_status_starts_running_then_succeeds(spec: TrainingJobSpec) -> None:
    backend = MockBackend(run_duration_s=0.05)
    handle = backend.submit(spec)
    status = backend.status(handle)
    assert status.status == JobStatus.RUNNING
    time.sleep(0.1)
    status = backend.status(handle)
    assert status.status == JobStatus.SUCCEEDED
    assert status.exit_code == 0


def test_mock_backend_cancel_makes_job_cancelled(spec: TrainingJobSpec) -> None:
    backend = MockBackend(run_duration_s=10.0)
    handle = backend.submit(spec)
    backend.cancel(handle)
    status = backend.status(handle)
    assert status.status == JobStatus.CANCELLED


def test_mock_backend_cancel_idempotent(spec: TrainingJobSpec) -> None:
    backend = MockBackend(run_duration_s=0.01)
    handle = backend.submit(spec)
    time.sleep(0.05)
    # already succeeded; cancel should be a no-op
    backend.cancel(handle)
    status = backend.status(handle)
    assert status.status == JobStatus.SUCCEEDED  # unchanged


def test_mock_backend_unavailable_mode_raises(spec: TrainingJobSpec) -> None:
    backend = MockBackend(unavailable=True)
    with pytest.raises(BackendUnavailableError):
        backend.submit(spec)


def test_mock_backend_rejects_too_many_gpus(spec: TrainingJobSpec) -> None:
    backend = MockBackend()
    bad_spec = TrainingJobSpec(
        job_id=spec.job_id,
        name=spec.name,
        tenant=spec.tenant,
        owner=spec.owner,
        code_image=spec.code_image,
        code_command=spec.code_command,
        resources=ResourceSpec(cpus=1, memory_gi=2, gpus=1000),
    )
    with pytest.raises(InvalidJobSpecError):
        backend.submit(bad_spec)


def test_status_for_unknown_handle_raises(spec: TrainingJobSpec) -> None:
    backend = MockBackend()
    bogus = JobHandle(backend="mock", native_id="does-not-exist", submitted_at=spec.metadata.__class__())  # type: ignore
    # quick & dirty: just construct a handle manually
    import datetime as _dt
    bogus = JobHandle(
        backend="mock",
        native_id="does-not-exist",
        submitted_at=_dt.datetime.now(_dt.timezone.utc),
    )
    with pytest.raises(JobNotFoundError):
        backend.status(bogus)


def test_registry_lists_registered_backends() -> None:
    registry = BackendRegistry()
    registry.register(MockBackend())
    assert registry.names() == ["mock"]


def test_registry_pick_returns_default_when_unspecified() -> None:
    registry = BackendRegistry()
    backend = MockBackend()
    registry.register(backend)
    assert registry.pick() is backend


def test_registry_pick_with_unknown_name_raises() -> None:
    registry = BackendRegistry()
    registry.register(MockBackend())
    with pytest.raises(KeyError):
        registry.pick(preferred="not-real")
```

**TODO**: Save this. If you have pytest, run `pytest tests/test_plugin.py -v`. Expected: all tests pass.

If you cannot run pytest, walk through the tests and confirm by inspection that they exercise the expected paths.

---

## Part 6: Operational Documentation (10 minutes)

Create `plugin/README.md` documenting the plugin system for platform operators and plugin authors.

```markdown
# Aurelia Training Backend Plugins

## Overview

The Aurelia platform dispatches submitted training jobs to one of several
*backends*. Each backend is a plugin that conforms to the
`TrainingBackend` protocol (`plugin/interface.py`).

This README is for two audiences:

- **Plugin authors** who want to write a new backend.
- **Platform operators** who run the platform with multiple backends.

## For Plugin Authors

### Conforming to the protocol

Your backend class must:

1. Have `name` and `description` class attributes.
2. Implement `submit`, `status`, `cancel`, `logs` methods.
3. Be safe to call from multiple threads.
4. Raise the standard exceptions on the standard error conditions:
   - `InvalidJobSpecError` if the spec cannot be run by this backend.
   - `BackendUnavailableError` if the backend itself is unreachable.
   - `JobNotFoundError` if a handle refers to nothing.

See `plugin/backends/mock.py` for a complete reference implementation.

### Distributing as a plugin

Your backend must be importable as a Python module. Declare it as an
entry point in your `pyproject.toml`:

    [project.entry-points."aurelia.training_backends"]
    my_backend = "my_pkg.module:MyBackend"

When the platform starts, `discover_backends_from_entry_points()` will
find and load your plugin.

### What your plugin can NOT do

By contract:

- Your plugin does not see user identity (only `spec.owner` and
  `spec.tenant` as strings; no tokens, no credentials).
- Your plugin does not enforce quota (the platform does).
- Your plugin does not bill (the platform does).
- Your plugin does not notify users (the platform does).
- Your plugin runs in the platform's process; it must not crash the
  process or it will take down the platform for all other tenants.

### What your plugin SHOULD do

- Log structured events at INFO level with sufficient context to debug
  failures.
- Return rich `backend_details` in `JobStatusReport` for the platform
  to surface.
- Be idempotent on cancellation (cancelling a finished job is a no-op).
- Be resilient to backend-side transient errors (retry with backoff on
  network blips; don't bubble every transient error up to the user).

## For Platform Operators

### Selecting backends

The platform picks a backend per-tenant via configuration:

    tenants:
      ml-research:
        backend: kubernetes
      voice-experiments:
        backend: vertex-ai
      dev-sandbox:
        backend: mock

If no backend is configured for a tenant, the platform's default
backend is used.

### Operational risks

Plugins are software you didn't write but are operationally
responsible for. Mitigations we use:

- **Plugin manifest**. Each plugin declares its version, owner, and
  resource expectations. We reject plugins with missing manifest fields.
- **Plugin observability**. Every plugin method invocation is wrapped
  in metrics (call rate, latency, error rate, error type).
- **Plugin failure budget**. If a plugin's error rate exceeds 5% for
  10 minutes, the platform stops routing new jobs to it and alerts the
  on-call.
- **Plugin deprecation**. Plugins go through the same deprecation
  workflow as APIs (see `_meta/PLAN.md` for the curriculum guardrails).

### Failure modes we accept

- A poorly-written plugin can block the platform's worker pool if it
  doesn't return from `status()` calls. Mitigation: enforce per-call
  timeouts via wrapping in the dispatcher.
- A plugin can leak memory (we have not solved this for in-process
  plugins; for tenants with very high-throughput requirements, we may
  move plugins to subprocess isolation).
- A plugin can have dependency conflicts with the platform itself
  (numpy versions, etc.). Mitigation: vendor common dependencies; pin
  conflicting versions in the platform.

### What the interface deliberately excludes

(From Lecture 05 — extension-point design.) The plugin interface
does not include:

- `get_metrics()` — metrics are platform-scoped.
- `set_priority()` — scheduling priority is platform-scoped.
- `get_quota_usage()` — quota tracking is platform-scoped.
- `notify_user()` — user notifications are platform-scoped.

Plugins should focus on *running jobs*. Anything beyond that is platform
territory.

## Roadmap

- Subprocess-based plugin isolation (in progress).
- Webhook-based plugins for cross-team extensions (planned).
- Plugin discovery via a central plugin registry (planned).
- Mandatory plugin signing (planned).
```

**TODO**: Save the README, adapting the content if your design differs.

---

## Part 7: Critique of Alternative Designs (10 minutes)

In `plugin/README.md` (or a separate `tradeoffs.md`), critique three alternative plugin designs.

### Alternative A: Pure subclassing (no Protocol)

Instead of structural typing via Protocol, require backend authors to subclass an abstract base class:

```python
class TrainingBackend(abc.ABC):
    @abc.abstractmethod
    def submit(self, spec: TrainingJobSpec) -> JobHandle: ...
    ...
```

**TODO**: Write 2-3 sentences. What does this gain (or lose)?

Hints (don't peek until tried):

- Gains: explicit "is-a" relationship; easier for IDEs; can include default implementations.
- Loses: forces inheritance, which is sometimes awkward when wrapping existing classes; harder for third-parties who want to wrap existing libraries.

### Alternative B: Subprocess plugins (JSON-RPC)

Instead of in-process Python, run each plugin as a separate process. The platform talks to it via stdin/stdout JSON-RPC.

**TODO**: Write 2-3 sentences. What does this gain (or lose)?

Hints:

- Gains: isolation (crashes don't propagate); language-agnostic; dependency isolation.
- Loses: IPC overhead; process management complexity; harder local development.

### Alternative C: Webhook plugins (external HTTP)

The plugin runs as a separate service somewhere on the network. The platform sends HTTP requests to it for each operation.

**TODO**: Write 2-3 sentences. What does this gain (or lose)?

Hints:

- Gains: total isolation; plugin can be operated by a different team; cross-org plugins possible.
- Loses: latency; failure modes (plugin down); authentication and authorization complexity.

### Your design's place

In `plugin/README.md`, write a paragraph: "Why we chose in-process Python over subprocess and webhook." Examples of reasoning:

- We have one team (the platform team) writing all backends — isolation isn't urgent.
- All backends are in Python — language-neutral isn't valuable.
- Performance matters; IPC overhead is real.
- Webhook plugins can be added later if cross-team plugins emerge as a need.
- Subprocess can be added later for hostile/external plugins.

---

## Common Pitfalls

1. **Letting plugins import too much platform-internal state.** If `MockBackend.__init__` is called with the entire platform's config, the plugin becomes tightly coupled to the platform. Keep plugin constructors narrow.
2. **No timeouts on plugin calls.** A plugin can hang. The dispatcher must enforce per-call timeouts.
3. **No metrics on plugin calls.** Without metrics, you'll discover plugin problems by user complaint.
4. **Letting plugins talk to each other.** This breeds tight coupling. Plugins should be independent; orchestration is the platform's job.
5. **Overly clever interface.** If the protocol has 17 methods, plugin authors can't realistically implement all of them. Keep the interface narrow (4 methods is typical).
6. **Forgetting thread-safety.** A platform dispatches many concurrent requests. Plugins must be thread-safe or the dispatcher must serialize them.

---

## Reflection Questions

Answer in `plugin/README.md` under a "Reflection" heading:

1. The protocol has 4 methods: `submit`, `status`, `cancel`, `logs`. Identify one method you'd add if asked by a backend author, and one you'd refuse. Defend each.
2. Plugins are loaded in-process. A future enterprise customer asks: "we want to write a plugin that we don't trust to share memory with your platform." What's your migration path?
3. The mock backend simulates job completion by elapsed time. Is that *too* clean — does it hide real failure modes the platform should be tested against? What would you add?
4. The `BackendRegistry.pick()` method returns "the first backend by name" if no preferred backend is given. Is alphabetical-by-name really the right default? Should the platform pick by load? By cost? By tenant policy?
5. If you were doing this exercise again, what would you change about the interface? Be specific.

---

## Self-Assessment

- [ ] Can I explain why a Protocol (not an ABC) is appropriate here, in one sentence?
- [ ] Can I list 4 things that are deliberately not in the plugin interface?
- [ ] Can I name 3 risks of in-process plugins, and 3 alternatives?
- [ ] Did my tests pass (or, by inspection, look correct)?
- [ ] Could I add a Vertex AI backend in 30 minutes given this interface?

If yes to all, you're done.

---

## Suggested Time Allocation

| Section | Time |
| --- | --- |
| Part 1: Interface | 25 min |
| Part 2: Mock Backend | 15 min |
| Part 3: Kubernetes Backend | 20 min |
| Part 4: Registry | 10 min |
| Part 5: Tests | 10 min |
| Part 6: Operational Docs | 10 min |
| Part 7: Critique | 10 min |
| **Total** | **100 min** |

---

## Where to Go from Here

You now have a working plugin system. In Module 04 (Training Infrastructure) you'll see real backends in depth. In Module 09 (Security) you'll revisit the plugin trust model in more detail. In a future exercise (Module 11 capstone) you may extend this with a Vertex AI or AWS Batch backend.

For now: push your `plugin/` directory and `tests/` to your fork. Move on to Exercise 05.
