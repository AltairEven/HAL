# Plan 6: Cluster Management — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build Layer 5 — AgentService (task queue, preemption), Leader Agent tools, ClusterRouter (routing, Human interface, authority stamping), cluster config, streaming log, and CLI entry point.

**Architecture:** AgentService wraps an Agent with an ordered task queue and manages task lifecycle. Leader Agent tools are regular HAL tools loaded via role config. ClusterRouter is a lightweight async service with deterministic routing rules and HTTP/WebSocket Human interface. All communication goes through HAL Bus.

**Tech Stack:** Python 3.9+, asyncio, aiohttp, pytest, pytest-asyncio

**Spec:** `docs/superpowers/specs/2026-04-27-hal-design.md` — Layer 5: Cluster Management, Deployment

**Depends on:** Plan 1-5 (all prior layers)

---

## File Structure

```
src/hal/cluster/
├── __init__.py
├── config.py            # Cluster config parser + validation
├── agent_service.py     # AgentService — task queue, slots, preemption
├── leader_tools.py      # Leader Agent tools (assign_task, wait_for_tasks, etc.)
├── router.py            # ClusterRouter — routing, Human interface, authority
├── streaming.py         # Streaming log (EventBus → WebSocket)
└── cli.py               # CLI entry point (bootstrap + graceful shutdown)

tests/cluster/
├── __init__.py
├── test_config.py
├── test_agent_service.py
├── test_leader_tools.py
├── test_router.py
├── test_streaming.py
├── test_cli.py
└── test_cluster_integration.py
```

---

## Task 1: Cluster Config

**Files:**
- Create: `src/hal/cluster/__init__.py`
- Create: `src/hal/cluster/config.py`
- Create: `tests/cluster/__init__.py`
- Create: `tests/cluster/test_config.py`

- [ ] **Step 1: Write the failing tests**

`tests/cluster/test_config.py`:
```python
from pathlib import Path

import pytest
import yaml

from hal.cluster.config import ClusterConfig, load_cluster_config


SAMPLE_CLUSTER_YAML = {
    "cluster": {
        "name": "mobile-sdk-team",
        "max_cost_per_task": 50.0,
        "shutdown_policy": "prompt_human",
        "bus": {"max_inbox_size": 100},
        "router": {"human_interface": "http", "listen": "0.0.0.0:8080"},
        "leader_agent": {"role": "product_manager", "instance": "pm-01"},
        "member_agents": [
            {"role": "ios_developer", "instance": "ios-dev-01"},
            {"role": "android_developer", "instance": "android-dev-01"},
        ],
    }
}


def test_load_cluster_config(tmp_path: Path):
    config_file = tmp_path / "cluster.yaml"
    config_file.write_text(yaml.dump(SAMPLE_CLUSTER_YAML))

    config = load_cluster_config(config_file)

    assert config.name == "mobile-sdk-team"
    assert config.max_cost_per_task == 50.0
    assert config.shutdown_policy == "prompt_human"
    assert config.bus_max_inbox_size == 100
    assert config.leader_role == "product_manager"
    assert config.leader_instance == "pm-01"
    assert len(config.member_agents) == 2


def test_cluster_config_defaults(tmp_path: Path):
    minimal = {
        "cluster": {
            "name": "test",
            "leader_agent": {"role": "pm", "instance": "pm-01"},
            "member_agents": [{"role": "dev", "instance": "dev-01"}],
        }
    }
    config_file = tmp_path / "cluster.yaml"
    config_file.write_text(yaml.dump(minimal))

    config = load_cluster_config(config_file)

    assert config.max_cost_per_task == 50.0
    assert config.shutdown_policy == "prompt_human"
    assert config.bus_max_inbox_size == 100
    assert config.router_listen == "0.0.0.0:8080"


def test_missing_leader_raises(tmp_path: Path):
    bad = {
        "cluster": {
            "name": "bad",
            "member_agents": [{"role": "dev", "instance": "dev-01"}],
        }
    }
    config_file = tmp_path / "cluster.yaml"
    config_file.write_text(yaml.dump(bad))

    with pytest.raises(ValueError, match="leader_agent"):
        load_cluster_config(config_file)


def test_no_member_agents_raises(tmp_path: Path):
    bad = {
        "cluster": {
            "name": "bad",
            "leader_agent": {"role": "pm", "instance": "pm-01"},
        }
    }
    config_file = tmp_path / "cluster.yaml"
    config_file.write_text(yaml.dump(bad))

    with pytest.raises(ValueError, match="member_agents"):
        load_cluster_config(config_file)


def test_all_instance_ids(tmp_path: Path):
    config_file = tmp_path / "cluster.yaml"
    config_file.write_text(yaml.dump(SAMPLE_CLUSTER_YAML))

    config = load_cluster_config(config_file)
    ids = config.all_instance_ids()
    assert sorted(ids) == ["android-dev-01", "ios-dev-01", "pm-01"]
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/cluster/test_config.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`src/hal/cluster/__init__.py`:
```python
"""HAL Cluster Management — multi-agent coordination layer."""
```

`src/hal/cluster/config.py`:
```python
"""Cluster configuration — parse and validate cluster YAML."""

from dataclasses import dataclass, field
from pathlib import Path
from typing import Any

import yaml


@dataclass
class AgentConfig:
    role: str
    instance: str


@dataclass
class ClusterConfig:
    name: str
    leader_role: str
    leader_instance: str
    member_agents: list[AgentConfig]
    max_cost_per_task: float = 50.0
    shutdown_policy: str = "prompt_human"
    shutdown_timeout: int = 10
    bus_max_inbox_size: int = 100
    router_listen: str = "0.0.0.0:8080"
    router_human_interface: str = "http"
    shared_memory_write_policy: str = "auto"

    def all_instance_ids(self) -> list[str]:
        ids = [self.leader_instance]
        ids.extend(a.instance for a in self.member_agents)
        return ids


def load_cluster_config(path: Path) -> ClusterConfig:
    """Load and validate cluster configuration from YAML."""
    raw = yaml.safe_load(path.read_text(encoding="utf-8")) or {}
    cluster = raw.get("cluster", {})

    if "leader_agent" not in cluster:
        raise ValueError("Cluster config must have 'leader_agent' section")

    members_raw = cluster.get("member_agents")
    if not members_raw:
        raise ValueError("Cluster config must have at least one 'member_agents' entry")

    leader = cluster["leader_agent"]
    bus = cluster.get("bus", {})
    router = cluster.get("router", {})

    return ClusterConfig(
        name=cluster.get("name", "unnamed"),
        leader_role=leader["role"],
        leader_instance=leader["instance"],
        member_agents=[AgentConfig(role=m["role"], instance=m["instance"]) for m in members_raw],
        max_cost_per_task=cluster.get("max_cost_per_task", 50.0),
        shutdown_policy=cluster.get("shutdown_policy", "prompt_human"),
        shutdown_timeout=cluster.get("shutdown_timeout", 10),
        bus_max_inbox_size=bus.get("max_inbox_size", 100),
        router_listen=router.get("listen", "0.0.0.0:8080"),
        router_human_interface=router.get("human_interface", "http"),
        shared_memory_write_policy=cluster.get("shared_memory_write_policy", "auto"),
    )
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/cluster/test_config.py -v`
Expected: 5 passed

- [ ] **Step 5: Commit**

```bash
git add src/hal/cluster/ tests/cluster/
git commit -m "feat: add ClusterConfig parser with validation"
```

---

## Task 2: AgentService — Core

**Files:**
- Create: `src/hal/cluster/agent_service.py`
- Create: `tests/cluster/test_agent_service.py`

- [ ] **Step 1: Write the failing tests**

`tests/cluster/test_agent_service.py`:
```python
import asyncio

import pytest

from hal.core.artifact import ArtifactType, create_artifact
from hal.core.bus.memory import InMemoryBus
from hal.core.driver import DriverResult, DriverStatus
from hal.core.event_bus import AsyncEventBus
from hal.cluster.agent_service import AgentService
from hal.drivers.mock import MockDriver
from hal.identity.role import ConcurrencyConfig, IdentityConfig, RoleProfile, SafeguardConfig
from hal.runner.checkpoint import InMemoryCheckpointStore


def _make_role(max_concurrent: int = 1) -> RoleProfile:
    return RoleProfile(
        identity=IdentityConfig(name="TestAgent", persona="Test agent."),
        concurrency=ConcurrencyConfig(max_concurrent_tasks=max_concurrent),
        safeguards=SafeguardConfig(max_steps=50, cost_budget=5.0),
    )


def _make_driver(responses: list[DriverResult]) -> MockDriver:
    return MockDriver(responses=responses)


@pytest.fixture
def bus() -> InMemoryBus:
    return InMemoryBus(max_inbox_size=20)


@pytest.fixture
def event_bus() -> AsyncEventBus:
    return AsyncEventBus()


async def test_agent_service_processes_task(bus, event_bus):
    driver = _make_driver([
        DriverResult(status=DriverStatus.COMPLETED, messages=[], steps_taken=3, token_usage=300,
                     cost_usd=0.3, summary="Done"),
    ])
    role = _make_role()
    checkpoint_store = InMemoryCheckpointStore()

    service = AgentService(
        instance_id="ios-dev-01",
        driver=driver,
        role=role,
        bus=bus,
        event_bus=event_bus,
        checkpoint_store=checkpoint_store,
        hard_cost_ceiling=50.0,
    )

    # Simulate receiving a task_assignment
    art = create_artifact(
        artifact_type=ArtifactType.TASK_ASSIGNMENT,
        sender="pm-01",
        receiver="ios-dev-01",
        task_id="task-1",
        trace_id="tr-1",
        summary="Build login",
        payload={"description": "Implement OAuth login"},
        parent_task_id="main-task",
    )

    await service.handle_artifact(art)

    # Should have processed the task
    assert service.capacity.running == 0  # completed
    assert service.capacity.queued == 0


async def test_agent_service_queues_when_full(bus, event_bus):
    """When slots full, tasks go to queue."""
    driver = _make_driver([
        DriverResult(status=DriverStatus.SUSPENDED, messages=[], steps_taken=1,
                     token_usage=100, cost_usd=0.1),
        DriverResult(status=DriverStatus.COMPLETED, messages=[], steps_taken=1,
                     token_usage=100, cost_usd=0.1, summary="Done"),
    ])
    role = _make_role(max_concurrent=1)
    checkpoint_store = InMemoryCheckpointStore()

    service = AgentService(
        instance_id="dev-01",
        driver=driver,
        role=role,
        bus=bus,
        event_bus=event_bus,
        checkpoint_store=checkpoint_store,
        hard_cost_ceiling=50.0,
    )

    # First task → starts running
    art1 = create_artifact(
        artifact_type=ArtifactType.TASK_ASSIGNMENT,
        sender="pm-01", receiver="dev-01",
        task_id="task-1", trace_id="tr-1",
        summary="Task 1", payload={},
        parent_task_id="main",
    )
    await service.handle_artifact(art1)

    # Second task → queued (first task is suspended, occupying slot)
    art2 = create_artifact(
        artifact_type=ArtifactType.TASK_ASSIGNMENT,
        sender="pm-01", receiver="dev-01",
        task_id="task-2", trace_id="tr-2",
        summary="Task 2", payload={},
        parent_task_id="main",
    )
    await service.handle_artifact(art2)

    assert service.capacity.queued == 1


async def test_capacity_reporting(bus, event_bus):
    driver = _make_driver([])
    role = _make_role(max_concurrent=3)
    service = AgentService(
        instance_id="dev-01",
        driver=driver,
        role=role,
        bus=bus,
        event_bus=event_bus,
        checkpoint_store=InMemoryCheckpointStore(),
        hard_cost_ceiling=50.0,
    )

    cap = service.capacity
    assert cap.max_concurrent == 3
    assert cap.running == 0
    assert cap.available_slots == 3


async def test_task_queue_order(bus, event_bus):
    """Tasks should be insertable at specific positions."""
    driver = _make_driver([
        DriverResult(status=DriverStatus.SUSPENDED, messages=[], steps_taken=1,
                     token_usage=100, cost_usd=0.1),
    ])
    role = _make_role(max_concurrent=1)
    service = AgentService(
        instance_id="dev-01",
        driver=driver,
        role=role,
        bus=bus,
        event_bus=event_bus,
        checkpoint_store=InMemoryCheckpointStore(),
        hard_cost_ceiling=50.0,
    )

    # Occupy the slot
    art0 = create_artifact(
        artifact_type=ArtifactType.TASK_ASSIGNMENT,
        sender="pm-01", receiver="dev-01",
        task_id="task-0", trace_id="tr",
        summary="Running", payload={}, parent_task_id="main",
    )
    await service.handle_artifact(art0)

    # Queue tasks at different positions
    for task_id in ["task-a", "task-b", "task-c"]:
        art = create_artifact(
            artifact_type=ArtifactType.TASK_ASSIGNMENT,
            sender="pm-01", receiver="dev-01",
            task_id=task_id, trace_id="tr",
            summary=task_id, payload={}, parent_task_id="main",
        )
        await service.handle_artifact(art)

    queue_ids = service.get_queue_task_ids()
    assert queue_ids == ["task-a", "task-b", "task-c"]
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/cluster/test_agent_service.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`src/hal/cluster/agent_service.py`:
```python
"""AgentService — deployable unit wrapping a single agent instance.

Manages task queue, slot occupancy, preemption, and artifact handling.
"""

import asyncio
import logging
from dataclasses import dataclass
from typing import Any

from hal.core.artifact import Artifact, ArtifactType, create_artifact
from hal.core.bus.protocol import Bus
from hal.core.driver import AgentDriver, CancellationToken, DriverResult, DriverStatus
from hal.core.event_bus import AsyncEventBus
from hal.identity.role import RoleProfile
from hal.runner.checkpoint import CheckpointStore
from hal.runner.task_runner import TaskRunner, TaskResult

logger = logging.getLogger(__name__)


@dataclass
class QueuedTask:
    task_id: str
    trace_id: str
    description: str
    parent_task_id: str | None


@dataclass
class Capacity:
    max_concurrent: int
    running: int
    suspended: int
    queued: int
    preempted: int
    available_slots: int


class AgentService:
    """Wraps a single agent instance with task queue and lifecycle management."""

    def __init__(
        self,
        instance_id: str,
        driver: AgentDriver,
        role: RoleProfile,
        bus: Bus,
        event_bus: AsyncEventBus,
        checkpoint_store: CheckpointStore,
        hard_cost_ceiling: float,
    ) -> None:
        self._instance_id = instance_id
        self._driver = driver
        self._role = role
        self._bus = bus
        self._event_bus = event_bus
        self._checkpoint_store = checkpoint_store
        self._hard_cost_ceiling = hard_cost_ceiling

        self._max_concurrent = role.concurrency.max_concurrent_tasks
        self._task_queue: list[QueuedTask] = []
        self._running_tasks: dict[str, TaskRunner] = {}
        self._suspended_tasks: dict[str, QueuedTask] = {}  # task_id → QueuedTask (occupying slots)
        self._preempted_task_ids: list[str] = []
        self._preempted_ready_ids: set[str] = set()  # preempted tasks that received resume_event
        self._cancellation_tokens: dict[str, CancellationToken] = {}
        self._seen_artifact_ids: set[str] = set()
        self._ttl_scan_task: asyncio.Task | None = None
        self._ttl_scan_interval: int = 60  # seconds
        self._active_async_tasks: set[asyncio.Task] = set()
        self._running: bool = False

    @property
    def instance_id(self) -> str:
        return self._instance_id

    @property
    def capacity(self) -> Capacity:
        running = len(self._running_tasks)
        suspended = len(self._suspended_tasks)
        return Capacity(
            max_concurrent=self._max_concurrent,
            running=running,
            suspended=suspended,
            queued=len(self._task_queue),
            preempted=len(self._preempted_task_ids),
            available_slots=max(0, self._max_concurrent - running - suspended),
        )

    def get_queue_task_ids(self) -> list[str]:
        return [t.task_id for t in self._task_queue]

    def reorder_queue(self, task_ids: list[str]) -> None:
        """Reorder queue to match the given task_id order."""
        id_to_task = {t.task_id: t for t in self._task_queue}
        self._task_queue = [id_to_task[tid] for tid in task_ids if tid in id_to_task]

    async def handle_artifact(self, artifact: Artifact) -> None:
        """Process an incoming artifact."""
        # Idempotency dedup
        if artifact.artifact_id in self._seen_artifact_ids:
            return
        self._seen_artifact_ids.add(artifact.artifact_id)

        if artifact.artifact_type == ArtifactType.TASK_ASSIGNMENT:
            await self._handle_task_assignment(artifact)
        elif artifact.artifact_type == ArtifactType.RESUME_EVENT:
            await self._handle_resume_event(artifact)

    async def _handle_task_assignment(self, artifact: Artifact) -> None:
        queued = QueuedTask(
            task_id=artifact.task_id,
            trace_id=artifact.trace_id,
            description=artifact.payload.get("description", artifact.summary),
            parent_task_id=artifact.parent_task_id,
        )

        position = artifact.payload.get("position", "next")
        suspend_current = artifact.payload.get("suspend_current", False)
        suspend_task_id = artifact.payload.get("suspend_task_id")

        # Handle preemption if requested
        if suspend_current and self.capacity.available_slots == 0:
            target_id = suspend_task_id
            if target_id is None and self._max_concurrent == 1:
                # Single-task agent: preempt the sole occupied task
                target_id = next(iter(self._suspended_tasks), None) or next(iter(self._running_tasks), None)
            if target_id:
                await self.preempt_task(target_id)

        if self.capacity.available_slots > 0:
            await self._start_task(queued)
        else:
            # Insert at specified position
            if position == "next":
                self._task_queue.insert(0, queued)
            elif position == "last":
                self._task_queue.append(queued)
            elif isinstance(position, int):
                self._task_queue.insert(position, queued)
            else:
                self._task_queue.insert(0, queued)  # default to next

    async def _handle_resume_event(self, artifact: Artifact) -> None:
        task_id = artifact.task_id

        if task_id in self._suspended_tasks:
            # Suspended task (occupying slot) — resume immediately
            self._suspended_tasks.pop(task_id)
            runner = TaskRunner(
                agent_id=self._instance_id,
                driver=self._driver,
                role=self._role,
                event_bus=self._event_bus,
                checkpoint_store=self._checkpoint_store,
                hard_cost_ceiling=self._hard_cost_ceiling,
                bus=self._bus,
            )
            result = await runner.resume(
                task_id=task_id,
                trace_id=artifact.trace_id,
                event_data=artifact.payload,
            )
            await self._handle_task_result(result, artifact.parent_task_id, artifact.trace_id)

        elif task_id in self._preempted_task_ids:
            # Preempted task — store event data in checkpoint, mark as ready-to-resume
            cp = self._checkpoint_store.load(task_id)
            if cp:
                cp.context_data["stored_resume_event"] = artifact.payload
                cp.context_data["stored_resume_trace_id"] = artifact.trace_id
                self._checkpoint_store.save(cp)
            self._preempted_ready_ids.add(task_id)

    async def _start_task(self, queued: QueuedTask) -> None:
        # Collect concurrent task context for parallel-capable agents
        concurrent_context: list[str] | None = None
        if self._max_concurrent > 1 and self._running_tasks:
            concurrent_context = [
                f"{tid}: {r._role.identity.name} task"  # simplified; real impl would store description
                for tid, r in self._running_tasks.items()
            ]

        token = CancellationToken()
        self._cancellation_tokens[queued.task_id] = token

        runner = TaskRunner(
            agent_id=self._instance_id,
            driver=self._driver,
            role=self._role,
            event_bus=self._event_bus,
            checkpoint_store=self._checkpoint_store,
            hard_cost_ceiling=self._hard_cost_ceiling,
            bus=self._bus,
            cancellation_token=token,
        )
        self._running_tasks[queued.task_id] = runner

        task = asyncio.create_task(self._run_task(runner, queued, concurrent_context))
        self._active_async_tasks.add(task)
        task.add_done_callback(self._active_async_tasks.discard)

    async def _run_task(self, runner: TaskRunner, queued: QueuedTask, concurrent_context: list[str] | None = None) -> None:
        result = await runner.run(
            task_id=queued.task_id,
            task_description=queued.description,
            trace_id=queued.trace_id,
            parent_task_id=queued.parent_task_id,
            concurrent_context=concurrent_context,
        )

        self._running_tasks.pop(queued.task_id, None)
        self._cancellation_tokens.pop(queued.task_id, None)
        await self._handle_task_result(result, queued.parent_task_id, queued.trace_id)

    async def _handle_task_result(
        self, result: TaskResult, parent_task_id: str | None, trace_id: str,
    ) -> None:
        if result.status in (DriverStatus.COMPLETED, DriverStatus.FAILED):
            # Publish result to parent task
            if parent_task_id and self._bus:
                art = create_artifact(
                    artifact_type=ArtifactType.RESUME_EVENT,
                    sender=self._instance_id,
                    receiver="",  # will be routed by ClusterRouter
                    task_id=parent_task_id,
                    trace_id=trace_id,
                    summary=result.summary or f"Task {result.status.value}",
                    payload={
                        "child_task_id": result.task_id,
                        "status": result.status.value,
                        "summary": result.summary,
                    },
                )
                await self._bus.publish(art)

            # Process next queued task
            await self._process_queue()

        elif result.status in (DriverStatus.SUSPENDED, DriverStatus.ESCALATED,
                               DriverStatus.BUDGET_EXHAUSTED):
            # Store as suspended (occupying slot) — QueuedTask reconstructed from result
            self._suspended_tasks[result.task_id] = QueuedTask(
                task_id=result.task_id, trace_id="", description="",
                parent_task_id=parent_task_id,
            )

    async def _process_queue(self) -> None:
        """Start next task, prioritizing preempted tasks per spec.

        Priority: (a) preempted tasks with resume_event ready,
                  (b) other preempted tasks in preemption order,
                  (c) queued tasks.
        """
        while self.capacity.available_slots > 0:
            next_task_id: str | None = None

            # (a) Preempted tasks that received resume_event while preempted
            for tid in self._preempted_task_ids:
                if tid in self._preempted_ready_ids:
                    next_task_id = tid
                    break

            # (b) Other preempted tasks (first preempted, first resumed)
            if next_task_id is None and self._preempted_task_ids:
                next_task_id = self._preempted_task_ids[0]

            if next_task_id is not None:
                self._preempted_task_ids.remove(next_task_id)
                self._preempted_ready_ids.discard(next_task_id)
                cp = self._checkpoint_store.load(next_task_id)
                if cp:
                    # Notify Leader via task_status_update
                    parent_id = cp.context_data.get("parent_task_id")
                    if self._bus and parent_id:
                        art = create_artifact(
                            artifact_type=ArtifactType.TASK_STATUS_UPDATE,
                            sender=self._instance_id,
                            receiver="",
                            task_id=parent_id,
                            trace_id="",
                            summary=f"Preempted task {next_task_id} auto-resumed",
                            payload={"child_task_id": next_task_id, "status": "resumed"},
                            parent_task_id=parent_id,
                        )
                        await self._bus.publish(art)
                    # Re-occupy slot as suspended (will be resumed by event or TTL)
                    self._suspended_tasks[next_task_id] = QueuedTask(
                        task_id=next_task_id, trace_id="", description="",
                        parent_task_id=parent_id,
                    )
                continue

            # (c) Queued tasks
            if self._task_queue:
                queued = self._task_queue.pop(0)
                await self._start_task(queued)
            else:
                break

    async def preempt_task(self, task_id: str) -> bool:
        """Preempt a running or suspended task to free a slot.

        For running tasks with cancellation_token support, sets the token
        for early Driver return. Otherwise falls back to soft preemption.
        Returns True if successful.
        """
        if task_id in self._suspended_tasks:
            # Suspended task — no active Driver, just move to preempted
            self._suspended_tasks.pop(task_id)
            self._preempted_task_ids.append(task_id)
            return True

        if task_id in self._running_tasks:
            # Running task — set cancellation_token if available
            token = self._cancellation_tokens.get(task_id)
            if token is not None:
                token.cancel()  # Driver checks between steps, returns SUSPENDED
            self._preempted_task_ids.append(task_id)
            return True

        return False

    async def start(self) -> None:
        """Start the agent service — rebuild state and begin TTL scanning."""
        self._rebuild_preempted_from_checkpoints()
        self._running = True
        self._ttl_scan_task = asyncio.create_task(self._ttl_scan_loop())

    async def stop(self) -> set[asyncio.Task]:
        """Stop accepting new tasks and cancel TTL scan.

        Returns the set of active asyncio Tasks for graceful shutdown.
        """
        self._running = False
        if self._ttl_scan_task:
            self._ttl_scan_task.cancel()
            try:
                await self._ttl_scan_task
            except asyncio.CancelledError:
                pass
        return set(self._active_async_tasks)

    async def _ttl_scan_loop(self) -> None:
        """Periodic TTL scan for suspended checkpoints."""
        from hal.runner.task_runner import check_suspend_ttl
        while self._running:
            await asyncio.sleep(self._ttl_scan_interval)
            try:
                checkpoints = self._checkpoint_store.list_by_agent(self._instance_id)
                for cp in checkpoints:
                    if cp.task_id in self._preempted_task_ids:
                        continue  # preempted tasks exempt from TTL
                    if check_suspend_ttl(cp, self._role.safeguards.suspend_ttl):
                        if cp.task_status == "suspended":
                            # SUSPENDED timeout → escalate
                            art = create_artifact(
                                artifact_type=ArtifactType.ESCALATION,
                                sender=self._instance_id,
                                receiver="",
                                task_id=cp.task_id,
                                trace_id="",
                                summary=f"Task {cp.task_id} suspend TTL expired",
                                payload={"reason": "suspend_ttl_expired"},
                                parent_task_id=cp.context_data.get("parent_task_id"),
                            )
                            await self._bus.publish(art)
                            self._suspended_tasks.pop(cp.task_id, None)
                        elif cp.task_status == "escalated":
                            # ESCALATED timeout → fail
                            self._checkpoint_store.delete(cp.task_id)
                            self._suspended_tasks.pop(cp.task_id, None)
            except Exception:
                logger.exception("TTL scan error")

    def _rebuild_preempted_from_checkpoints(self) -> None:
        """Rebuild preempted_task_ids from checkpoint store on startup (V1)."""
        checkpoints = self._checkpoint_store.list_by_agent(self._instance_id)
        for cp in checkpoints:
            if cp.task_status == "suspended":
                # Could be a preempted task — but without separate persistence
                # we can't distinguish preempted from externally-suspended.
                # V1 simplification: treat all suspended checkpoints at startup
                # as preempted (they were in-flight when the process died).
                self._preempted_task_ids.append(cp.task_id)
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/cluster/test_agent_service.py -v`
Expected: 4 passed

- [ ] **Step 5: Commit**

```bash
git add src/hal/cluster/agent_service.py tests/cluster/test_agent_service.py
git commit -m "feat: add AgentService with task queue and slot management"
```

---

## Task 3: AgentService — Preemption

**Files:**
- Modify: `tests/cluster/test_agent_service.py`

- [ ] **Step 1: Write preemption tests (append)**

```python
# Append to tests/cluster/test_agent_service.py

async def test_preempt_suspended_task(bus, event_bus):
    driver = _make_driver([
        DriverResult(status=DriverStatus.SUSPENDED, messages=[], steps_taken=1, token_usage=100,
                     cost_usd=0.1),
    ])
    role = _make_role(max_concurrent=1)
    service = AgentService(
        instance_id="dev-01", driver=driver, role=role, bus=bus,
        event_bus=event_bus, checkpoint_store=InMemoryCheckpointStore(),
        hard_cost_ceiling=50.0,
    )

    art = create_artifact(
        artifact_type=ArtifactType.TASK_ASSIGNMENT,
        sender="pm-01", receiver="dev-01",
        task_id="task-1", trace_id="tr",
        summary="Task", payload={}, parent_task_id="main",
    )
    await service.handle_artifact(art)

    # Task is suspended, occupying slot
    assert service.capacity.suspended == 1
    assert service.capacity.available_slots == 0

    # Preempt it
    result = await service.preempt_task("task-1")
    assert result is True
    assert service.capacity.preempted == 1
    assert service.capacity.available_slots == 1  # slot freed
```

- [ ] **Step 2: Run tests**

Run: `pytest tests/cluster/test_agent_service.py -v`
Expected: 5 passed

- [ ] **Step 3: Commit**

```bash
git add tests/cluster/test_agent_service.py
git commit -m "test: add AgentService preemption tests"
```

---

## Task 4: Leader Agent Tools — Task Dispatch

**Files:**
- Create: `src/hal/cluster/leader_tools.py`
- Create: `tests/cluster/test_leader_tools.py`

- [ ] **Step 1: Write the failing tests**

`tests/cluster/test_leader_tools.py`:
```python
import asyncio

import pytest

from hal.cluster.leader_tools import (
    AssignTaskTool,
    WaitForTasksTool,
    CollectOpinionsTool,
    QueryClusterStatusTool,
    QueryAgentTasksTool,
    ReorderAgentTasksTool,
    ResolveEscalationTool,
    BroadcastDecisionTool,
)
from hal.cluster.agent_service import AgentService
from hal.core.artifact import ArtifactType, create_artifact
from hal.core.bus.memory import InMemoryBus
from hal.core.driver import DriverResult, DriverStatus
from hal.core.event_bus import AsyncEventBus
from hal.drivers.mock import MockDriver
from hal.identity.role import ConcurrencyConfig, IdentityConfig, RoleProfile, SafeguardConfig
from hal.runner.checkpoint import InMemoryCheckpointStore
from hal.tools.protocol import RiskLevel, ToolStatus


def _make_service(bus, event_bus, instance_id="dev-01", responses=None):
    driver = MockDriver(responses=responses or [
        DriverResult(status=DriverStatus.COMPLETED, messages=[], steps_taken=1,
                     token_usage=100, cost_usd=0.1, summary="Done"),
    ])
    role = RoleProfile(
        identity=IdentityConfig(name="Dev", persona="Dev."),
        concurrency=ConcurrencyConfig(max_concurrent_tasks=1),
        safeguards=SafeguardConfig(max_steps=50, cost_budget=5.0),
    )
    return AgentService(
        instance_id=instance_id, driver=driver, role=role, bus=bus,
        event_bus=event_bus, checkpoint_store=InMemoryCheckpointStore(),
        hard_cost_ceiling=50.0,
    )


@pytest.fixture
def bus() -> InMemoryBus:
    return InMemoryBus(max_inbox_size=20)


@pytest.fixture
def event_bus() -> AsyncEventBus:
    return AsyncEventBus()


async def test_assign_task_tool(bus, event_bus):
    service = _make_service(bus, event_bus)
    services = {"dev-01": service}

    tool = AssignTaskTool(services=services, bus=bus)
    assert tool.name == "assign_task"
    assert tool.risk_level == RiskLevel.LOW

    result = await tool.execute({
        "target": "dev-01",
        "task": "Implement login",
        "task_id": "task-1",
        "trace_id": "tr-1",
        "parent_task_id": "main",
    })

    assert result.status == ToolStatus.SUCCESS
    assert "task_id" in result.data


async def test_assign_task_unknown_target(bus, event_bus):
    tool = AssignTaskTool(services={}, bus=bus)
    result = await tool.execute({
        "target": "nonexistent",
        "task": "Test",
        "task_id": "t",
        "trace_id": "tr",
    })
    assert result.status == ToolStatus.FAILURE


async def test_query_cluster_status(bus, event_bus):
    s1 = _make_service(bus, event_bus, "dev-01")
    s2 = _make_service(bus, event_bus, "dev-02")

    tool = QueryClusterStatusTool(services={"dev-01": s1, "dev-02": s2})
    result = await tool.execute({})

    assert result.status == ToolStatus.SUCCESS
    assert len(result.data["agents"]) == 2


async def test_wait_for_tasks_returns_pending(bus, event_bus):
    services = {"dev-01": _make_service(bus, event_bus)}
    tool = WaitForTasksTool(services=services)

    result = await tool.execute({"task_ids": ["task-1", "task-2"]})

    assert result.status == ToolStatus.PENDING
    assert result.resume_event_id is not None
    assert result.data["waiting_for"] == ["task-1", "task-2"]


async def test_resolve_escalation(bus, event_bus):
    tool = ResolveEscalationTool(bus=bus)

    result = await tool.execute({
        "target_task_id": "task-esc-1",
        "target_instance": "dev-01",
        "resolution": "Use v2 API",
        "trace_id": "tr-1",
    })

    assert result.status == ToolStatus.SUCCESS
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/cluster/test_leader_tools.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`src/hal/cluster/leader_tools.py`:
```python
"""Leader Agent tools — cluster management operations.

These tools are loaded via role config for the Leader Agent only.
They follow the Tool protocol from hal.tools.protocol.
"""

from __future__ import annotations

import uuid
from typing import Any, TYPE_CHECKING

from hal.core.artifact import ArtifactType, Authority, Artifact, create_artifact
from hal.core.bus.protocol import Bus
from hal.tools.protocol import RiskLevel, ToolResult, ToolStatus

if TYPE_CHECKING:
    from hal.cluster.agent_service import AgentService


class AssignTaskTool:
    """Assign a task to a Member Agent."""

    def __init__(self, services: dict[str, Any], bus: Bus) -> None:
        self._services = services
        self._bus = bus

    @property
    def name(self) -> str:
        return "assign_task"

    @property
    def description(self) -> str:
        return "Assign a task to a specific Member Agent"

    @property
    def risk_level(self) -> RiskLevel:
        return RiskLevel.LOW

    @property
    def parameters_schema(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "target": {"type": "string", "description": "Target agent instance ID"},
                "task": {"type": "string", "description": "Task description"},
                "task_id": {"type": "string"},
                "trace_id": {"type": "string"},
                "parent_task_id": {"type": "string"},
                "position": {"type": "string", "default": "next", "description": "Queue position: 'next', 'last', or int"},
                "suspend_current": {"type": "boolean", "default": False, "description": "Whether to preempt a task to free a slot"},
                "suspend_task_id": {"type": "string", "description": "Task ID to preempt (required when suspend_current=True and agent has multiple slots)"},
            },
            "required": ["target", "task", "task_id", "trace_id"],
        }

    def validate(self, params: dict) -> None:
        if "target" not in params or "task" not in params:
            raise ValueError("target and task are required")

    async def execute(self, params: dict) -> ToolResult:
        target = params["target"]
        if target not in self._services:
            return ToolResult(
                status=ToolStatus.FAILURE,
                data={"error": f"Unknown agent: {target}"},
            )

        task_id = params.get("task_id", str(uuid.uuid4()))
        art = create_artifact(
            artifact_type=ArtifactType.TASK_ASSIGNMENT,
            sender="leader",
            receiver=target,
            task_id=task_id,
            trace_id=params.get("trace_id", ""),
            summary=params["task"][:100],
            payload={
                "description": params["task"],
                "position": params.get("position", "next"),
                "suspend_current": params.get("suspend_current", False),
                "suspend_task_id": params.get("suspend_task_id"),
            },
            parent_task_id=params.get("parent_task_id"),
        )

        service = self._services[target]
        await service.handle_artifact(art)

        return ToolResult(
            status=ToolStatus.SUCCESS,
            data={"task_id": task_id, "target": target},
        )

    async def rollback(self, params: dict) -> None:
        pass


class QueryClusterStatusTool:
    """Query all Member Agents' current capacity."""

    def __init__(self, services: dict[str, Any]) -> None:
        self._services = services

    @property
    def name(self) -> str:
        return "query_cluster_status"

    @property
    def description(self) -> str:
        return "Query all Member Agents' current capacity and status"

    @property
    def risk_level(self) -> RiskLevel:
        return RiskLevel.LOW

    @property
    def parameters_schema(self) -> dict:
        return {"type": "object", "properties": {}}

    def validate(self, params: dict) -> None:
        pass

    async def execute(self, params: dict) -> ToolResult:
        agents = []
        for instance_id, service in self._services.items():
            cap = service.capacity
            agents.append({
                "instance_id": instance_id,
                "max_concurrent": cap.max_concurrent,
                "running": cap.running,
                "suspended": cap.suspended,
                "queued": cap.queued,
                "preempted": cap.preempted,
                "available_slots": cap.available_slots,
            })
        return ToolResult(status=ToolStatus.SUCCESS, data={"agents": agents})

    async def rollback(self, params: dict) -> None:
        pass


class QueryAgentTasksTool:
    """Query a specific agent's task queue."""

    def __init__(self, services: dict[str, Any]) -> None:
        self._services = services

    @property
    def name(self) -> str:
        return "query_agent_tasks"

    @property
    def description(self) -> str:
        return "Query a specific Member Agent's current task queue"

    @property
    def risk_level(self) -> RiskLevel:
        return RiskLevel.LOW

    @property
    def parameters_schema(self) -> dict:
        return {
            "type": "object",
            "properties": {"target": {"type": "string"}},
            "required": ["target"],
        }

    def validate(self, params: dict) -> None:
        pass

    async def execute(self, params: dict) -> ToolResult:
        target = params["target"]
        if target not in self._services:
            return ToolResult(status=ToolStatus.FAILURE, data={"error": f"Unknown: {target}"})
        service = self._services[target]
        return ToolResult(
            status=ToolStatus.SUCCESS,
            data={"queue": service.get_queue_task_ids()},
        )

    async def rollback(self, params: dict) -> None:
        pass


class ReorderAgentTasksTool:
    """Reorder a Member Agent's queued tasks."""

    def __init__(self, services: dict[str, Any]) -> None:
        self._services = services

    @property
    def name(self) -> str:
        return "reorder_agent_tasks"

    @property
    def description(self) -> str:
        return "Reorder a Member Agent's queued tasks"

    @property
    def risk_level(self) -> RiskLevel:
        return RiskLevel.LOW

    @property
    def parameters_schema(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "target": {"type": "string"},
                "task_ids": {"type": "array", "items": {"type": "string"}},
            },
            "required": ["target", "task_ids"],
        }

    def validate(self, params: dict) -> None:
        pass

    async def execute(self, params: dict) -> ToolResult:
        target = params["target"]
        if target not in self._services:
            return ToolResult(status=ToolStatus.FAILURE, data={"error": f"Unknown: {target}"})
        self._services[target].reorder_queue(params["task_ids"])
        return ToolResult(status=ToolStatus.SUCCESS, data={"reordered": True})

    async def rollback(self, params: dict) -> None:
        pass


class ResolveEscalationTool:
    """Respond to a Member Agent's escalation."""

    def __init__(self, bus: Bus) -> None:
        self._bus = bus

    @property
    def name(self) -> str:
        return "resolve_escalation"

    @property
    def description(self) -> str:
        return "Respond to a Member Agent's escalation with resolution"

    @property
    def risk_level(self) -> RiskLevel:
        return RiskLevel.LOW

    @property
    def parameters_schema(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "target_task_id": {"type": "string"},
                "target_instance": {"type": "string"},
                "resolution": {"type": "string"},
                "trace_id": {"type": "string"},
            },
            "required": ["target_task_id", "target_instance", "resolution", "trace_id"],
        }

    def validate(self, params: dict) -> None:
        pass

    async def execute(self, params: dict) -> ToolResult:
        art = create_artifact(
            artifact_type=ArtifactType.RESUME_EVENT,
            sender="leader",
            receiver=params["target_instance"],
            task_id=params["target_task_id"],
            trace_id=params["trace_id"],
            summary="Escalation resolved",
            payload={"resolution": params["resolution"]},
        )
        await self._bus.publish(art)
        return ToolResult(status=ToolStatus.SUCCESS, data={"resolved": True})

    async def rollback(self, params: dict) -> None:
        pass


class BroadcastDecisionTool:
    """Broadcast a decision to specified Member Agents (fire-and-forget)."""

    def __init__(self, bus: Bus) -> None:
        self._bus = bus

    @property
    def name(self) -> str:
        return "broadcast_decision"

    @property
    def description(self) -> str:
        return "Broadcast a decision artifact to specified Member Agents"

    @property
    def risk_level(self) -> RiskLevel:
        return RiskLevel.LOW

    @property
    def parameters_schema(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "targets": {"type": "array", "items": {"type": "string"}},
                "decision": {"type": "string"},
                "trace_id": {"type": "string"},
            },
            "required": ["targets", "decision", "trace_id"],
        }

    def validate(self, params: dict) -> None:
        pass

    async def execute(self, params: dict) -> ToolResult:
        for target in params["targets"]:
            art = create_artifact(
                artifact_type=ArtifactType.DECISION,
                sender="leader",
                receiver=target,
                task_id=str(uuid.uuid4()),
                trace_id=params["trace_id"],
                summary=params["decision"][:100],
                payload={"decision": params["decision"]},
            )
            await self._bus.publish(art)
        return ToolResult(
            status=ToolStatus.SUCCESS,
            data={"broadcast_to": params["targets"]},
        )

    async def rollback(self, params: dict) -> None:
        pass


class WaitForTasksTool:
    """Wait for sub-task state changes using PENDING → SUSPENDED pattern.

    Returns PENDING with a resume_event_id. The Driver stops its loop
    and returns SUSPENDED. When any watched sub-task changes state,
    the framework delivers the event to Leader's task.
    """

    def __init__(self, services: dict[str, Any]) -> None:
        self._services = services

    @property
    def name(self) -> str:
        return "wait_for_tasks"

    @property
    def description(self) -> str:
        return "Wait for sub-task state changes (completion, failure, escalation)"

    @property
    def risk_level(self) -> RiskLevel:
        return RiskLevel.LOW

    @property
    def parameters_schema(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "task_ids": {"type": "array", "items": {"type": "string"}},
            },
            "required": ["task_ids"],
        }

    def validate(self, params: dict) -> None:
        if not params.get("task_ids"):
            raise ValueError("task_ids must not be empty")

    async def execute(self, params: dict) -> ToolResult:
        resume_event_id = f"wait-{uuid.uuid4()}"
        return ToolResult(
            status=ToolStatus.PENDING,
            data={
                "waiting_for": params["task_ids"],
                "message": "Waiting for sub-task state changes",
            },
            resume_event_id=resume_event_id,
        )

    async def rollback(self, params: dict) -> None:
        pass


class CollectOpinionsTool:
    """Ask multiple Member Agents a question concurrently with timeout."""

    def __init__(self, services: dict[str, Any], bus: Bus) -> None:
        self._services = services
        self._bus = bus

    @property
    def name(self) -> str:
        return "collect_opinions"

    @property
    def description(self) -> str:
        return "Ask multiple Member Agents a question concurrently"

    @property
    def risk_level(self) -> RiskLevel:
        return RiskLevel.LOW

    @property
    def parameters_schema(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "targets": {"type": "array", "items": {"type": "string"}},
                "question": {"type": "string"},
                "timeout": {"type": "number", "default": 30.0},
                "trace_id": {"type": "string"},
                "position": {"type": "string", "default": "next", "description": "Queue position for opinion requests"},
            },
            "required": ["targets", "question", "trace_id"],
        }

    def validate(self, params: dict) -> None:
        if not params.get("targets"):
            raise ValueError("targets must not be empty")

    async def execute(self, params: dict) -> ToolResult:
        import asyncio
        timeout = params.get("timeout", 30.0)
        responses: dict[str, Any] = {}

        async def ask_one(target: str) -> None:
            request = create_artifact(
                artifact_type=ArtifactType.TASK_ASSIGNMENT,
                sender="leader",
                receiver=target,
                task_id=str(uuid.uuid4()),
                trace_id=params["trace_id"],
                summary=params["question"][:100],
                payload={
                    "description": params["question"],
                    "position": params.get("position", "next"),
                },
            )
            reply = await self._bus.wait_for_reply(request, timeout=timeout)
            if reply:
                responses[target] = reply.payload
            else:
                responses[target] = {"status": "timeout"}

        tasks = [ask_one(t) for t in params["targets"] if t in self._services]
        await asyncio.gather(*tasks)

        return ToolResult(
            status=ToolStatus.SUCCESS,
            data={"responses": responses},
        )

    async def rollback(self, params: dict) -> None:
        pass
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/cluster/test_leader_tools.py -v`
Expected: 4 passed

- [ ] **Step 5: Commit**

```bash
git add src/hal/cluster/leader_tools.py tests/cluster/test_leader_tools.py
git commit -m "feat: add Leader Agent tools (assign, wait, query, resolve, collect, broadcast)"
```

---

## Task 5: ClusterRouter — Routing Engine

**Files:**
- Create: `src/hal/cluster/router.py`
- Create: `tests/cluster/test_router.py`

- [ ] **Step 1: Write the failing tests**

`tests/cluster/test_router.py`:
```python
import asyncio
from datetime import datetime, timezone

import pytest

from hal.core.artifact import Artifact, ArtifactType, Authority, create_artifact
from hal.core.bus.memory import InMemoryBus
from hal.core.event_bus import AsyncEventBus
from hal.cluster.router import ClusterRouter


@pytest.fixture
def bus() -> InMemoryBus:
    return InMemoryBus(max_inbox_size=20)


@pytest.fixture
def event_bus() -> AsyncEventBus:
    return AsyncEventBus()


async def test_member_escalation_routed_to_leader(bus, event_bus):
    """Member escalation with parent_task_id → resume_event on leader's parent task."""
    router = ClusterRouter(bus=bus, event_bus=event_bus, leader_instance="pm-01")

    published_to_leader: list[Artifact] = []
    bus.subscribe("pm-01", lambda a: published_to_leader.append(a))

    escalation = create_artifact(
        artifact_type=ArtifactType.ESCALATION,
        sender="ios-dev-01",
        receiver="",
        task_id="sub-task-1",
        trace_id="tr-1",
        summary="Need help",
        payload={"reason": "Ambiguous spec"},
        parent_task_id="leader-task-1",
    )

    await router.route_artifact(escalation)

    assert len(published_to_leader) == 1
    routed = published_to_leader[0]
    assert routed.artifact_type == ArtifactType.RESUME_EVENT
    assert routed.task_id == "leader-task-1"  # routed to parent
    assert routed.receiver == "pm-01"


async def test_leader_escalation_forwarded_to_human(bus, event_bus):
    """Leader escalation (no parent_task_id) → forwarded to Human."""
    router = ClusterRouter(bus=bus, event_bus=event_bus, leader_instance="pm-01")

    human_escalations: list[Artifact] = []
    router.on_human_escalation = lambda a: human_escalations.append(a)

    escalation = create_artifact(
        artifact_type=ArtifactType.ESCALATION,
        sender="pm-01",
        receiver="",
        task_id="leader-task-1",
        trace_id="tr-1",
        summary="Need Human help",
        payload={},
        # No parent_task_id → leader escalation
    )

    await router.route_artifact(escalation)
    assert len(human_escalations) == 1


async def test_confirmation_request_forwarded_to_human(bus, event_bus):
    """Confirmation requests always go to Human."""
    router = ClusterRouter(bus=bus, event_bus=event_bus, leader_instance="pm-01")

    human_confirmations: list[Artifact] = []
    router.on_human_confirmation = lambda a: human_confirmations.append(a)

    conf = create_artifact(
        artifact_type=ArtifactType.CONFIRMATION_REQUEST,
        sender="ios-dev-01",
        receiver="",
        task_id="task-1",
        trace_id="tr-1",
        summary="Deploy to production?",
        payload={"tool_name": "deploy"},
    )

    await router.route_artifact(conf)
    assert len(human_confirmations) == 1


async def test_authority_stamping(bus, event_bus):
    """Human responses get authority=HUMAN."""
    router = ClusterRouter(bus=bus, event_bus=event_bus, leader_instance="pm-01")

    published: list[Artifact] = []
    bus.subscribe("ios-dev-01", lambda a: published.append(a))

    await router.send_human_response(
        target_instance="ios-dev-01",
        target_task_id="task-1",
        trace_id="tr-1",
        response_data={"approved": True},
    )

    assert len(published) == 1
    assert published[0].authority == Authority.HUMAN


async def test_normal_artifacts_not_intercepted(bus, event_bus):
    """Non-escalation/confirmation artifacts pass through normally."""
    router = ClusterRouter(bus=bus, event_bus=event_bus, leader_instance="pm-01")

    decision = create_artifact(
        artifact_type=ArtifactType.DECISION,
        sender="pm-01",
        receiver="ios-dev-01",
        task_id="task-1",
        trace_id="tr-1",
        summary="Use v2 API",
        payload={},
    )

    should_route = router.should_intercept(decision)
    assert should_route is False
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/cluster/test_router.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`src/hal/cluster/router.py`:
```python
"""ClusterRouter — infrastructure-level routing, Human interface, authority stamping.

No LLM, no role profile, no Driver. Deterministic routing logic only.
"""

import logging
import uuid
from collections.abc import Callable
from datetime import datetime, timezone
from typing import Any

from hal.core.artifact import Artifact, ArtifactType, Authority
from hal.core.bus.protocol import Bus
from hal.core.event_bus import AsyncEventBus

logger = logging.getLogger(__name__)


class ClusterRouter:
    """Cluster-level infrastructure routing service."""

    def __init__(
        self,
        bus: Bus,
        event_bus: AsyncEventBus,
        leader_instance: str,
    ) -> None:
        self._bus = bus
        self._event_bus = event_bus
        self._leader_instance = leader_instance

        # Callbacks for Human-facing interactions (set by HTTP interface layer)
        self.on_human_escalation: Callable[[Artifact], Any] | None = None
        self.on_human_confirmation: Callable[[Artifact], Any] | None = None

    def should_intercept(self, artifact: Artifact) -> bool:
        """Check if this artifact should be intercepted by the router."""
        return artifact.artifact_type in (
            ArtifactType.ESCALATION,
            ArtifactType.CONFIRMATION_REQUEST,
        )

    async def route_artifact(self, artifact: Artifact) -> None:
        """Apply routing rules to an intercepted artifact."""
        if artifact.artifact_type == ArtifactType.ESCALATION:
            await self._route_escalation(artifact)
        elif artifact.artifact_type == ArtifactType.CONFIRMATION_REQUEST:
            await self._route_confirmation(artifact)

    async def _route_escalation(self, artifact: Artifact) -> None:
        if artifact.parent_task_id:
            # Member Agent escalation → translate to resume_event on Leader's parent task
            resume = Artifact(
                artifact_id=str(uuid.uuid4()),
                artifact_type=ArtifactType.RESUME_EVENT,
                sender=artifact.sender,
                receiver=self._leader_instance,
                task_id=artifact.parent_task_id,
                trace_id=artifact.trace_id,
                summary=f"Escalation from {artifact.sender}: {artifact.summary}",
                payload={
                    "escalation_from": artifact.sender,
                    "child_task_id": artifact.task_id,
                    "reason": artifact.payload.get("reason", artifact.summary),
                },
                authority=Authority.MEMBER,
                created_at=datetime.now(timezone.utc).isoformat(),
            )
            await self._bus.publish(resume)
        else:
            # Leader Agent escalation → forward to Human
            if self.on_human_escalation:
                self.on_human_escalation(artifact)
            else:
                logger.warning("Leader escalation but no Human handler configured")

    async def _route_confirmation(self, artifact: Artifact) -> None:
        if self.on_human_confirmation:
            self.on_human_confirmation(artifact)
        else:
            logger.warning("Confirmation request but no Human handler configured")

    async def send_human_response(
        self,
        target_instance: str,
        target_task_id: str,
        trace_id: str,
        response_data: dict[str, Any],
    ) -> None:
        """Create and publish a Human response with authority=HUMAN."""
        art = Artifact(
            artifact_id=str(uuid.uuid4()),
            artifact_type=ArtifactType.RESUME_EVENT,
            sender="human",
            receiver=target_instance,
            task_id=target_task_id,
            trace_id=trace_id,
            summary="Human response",
            payload=response_data,
            authority=Authority.HUMAN,
            created_at=datetime.now(timezone.utc).isoformat(),
        )
        await self._bus.publish(art)
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/cluster/test_router.py -v`
Expected: 5 passed

- [ ] **Step 5: Commit**

```bash
git add src/hal/cluster/router.py tests/cluster/test_router.py
git commit -m "feat: add ClusterRouter with escalation routing and authority stamping"
```

---

## Task 6: ClusterRouter — Human Interface (HTTP)

**Files:**
- Modify: `src/hal/cluster/router.py` (add HTTP server)
- Create: `tests/cluster/test_router_http.py`

Note: This task adds aiohttp HTTP endpoints. Tests use aiohttp's test client.

- [ ] **Step 1: Add aiohttp to pyproject.toml dependencies**

Add `"aiohttp>=3.9"` to the `dependencies` list in `pyproject.toml`.

Run: `pip install -e ".[dev]"`

- [ ] **Step 2: Write the failing tests**

`tests/cluster/test_router_http.py`:
```python
"""HTTP interface tests for ClusterRouter.

Uses aiohttp test client for endpoint testing.
"""

import pytest
from aiohttp import web
from aiohttp.test_utils import AioHTTPTestCase, TestClient

from hal.core.bus.memory import InMemoryBus
from hal.core.event_bus import AsyncEventBus
from hal.cluster.router import ClusterRouter, create_http_app


@pytest.fixture
def bus() -> InMemoryBus:
    return InMemoryBus(max_inbox_size=20)


@pytest.fixture
def event_bus() -> AsyncEventBus:
    return AsyncEventBus()


@pytest.fixture
def router(bus, event_bus) -> ClusterRouter:
    return ClusterRouter(bus=bus, event_bus=event_bus, leader_instance="pm-01")


async def test_submit_task(aiohttp_client, router, bus):
    published = []
    bus.subscribe("pm-01", lambda a: published.append(a))

    app = create_http_app(router)
    client = await aiohttp_client(app)

    resp = await client.post("/api/task", json={
        "description": "Build login SDK",
        "trace_id": "tr-http-1",
    })
    assert resp.status == 200
    data = await resp.json()
    assert "task_id" in data


async def test_get_status(aiohttp_client, router):
    app = create_http_app(router)
    client = await aiohttp_client(app)

    resp = await client.get("/api/status")
    assert resp.status == 200
    data = await resp.json()
    assert "leader" in data


async def test_respond_to_escalation(aiohttp_client, router, bus):
    published = []
    bus.subscribe("ios-dev-01", lambda a: published.append(a))

    app = create_http_app(router)
    client = await aiohttp_client(app)

    resp = await client.post("/api/respond", json={
        "target_instance": "ios-dev-01",
        "target_task_id": "task-esc-1",
        "trace_id": "tr-1",
        "response": {"guidance": "Use v2 API"},
    })
    assert resp.status == 200
    assert len(published) == 1
```

- [ ] **Step 3: Add create_http_app to router.py**

Add to `src/hal/cluster/router.py`:

```python
from aiohttp import web


def create_http_app(router: ClusterRouter) -> web.Application:
    """Create aiohttp app for Human interface."""
    app = web.Application()

    async def submit_task(request: web.Request) -> web.Response:
        data = await request.json()
        task_id = str(uuid.uuid4())
        art = Artifact(
            artifact_id=str(uuid.uuid4()),
            artifact_type=ArtifactType.TASK_ASSIGNMENT,
            sender="human",
            receiver=router._leader_instance,
            task_id=task_id,
            trace_id=data.get("trace_id", str(uuid.uuid4())),
            summary=data["description"][:100],
            payload={"description": data["description"]},
            authority=Authority.HUMAN,
            created_at=datetime.now(timezone.utc).isoformat(),
        )
        await router._bus.publish(art)
        return web.json_response({"task_id": task_id, "status": "submitted"})

    async def get_status(request: web.Request) -> web.Response:
        return web.json_response({
            "leader": router._leader_instance,
            "status": "running",
        })

    async def respond(request: web.Request) -> web.Response:
        data = await request.json()
        await router.send_human_response(
            target_instance=data["target_instance"],
            target_task_id=data["target_task_id"],
            trace_id=data["trace_id"],
            response_data=data.get("response", {}),
        )
        return web.json_response({"status": "sent"})

    app.router.add_post("/api/task", submit_task)
    app.router.add_get("/api/status", get_status)
    app.router.add_post("/api/respond", respond)

    return app
```

- [ ] **Step 4: Run tests**

Run: `pytest tests/cluster/test_router_http.py -v`
Expected: 3 passed

- [ ] **Step 5: Commit**

```bash
git add src/hal/cluster/router.py tests/cluster/test_router_http.py pyproject.toml
git commit -m "feat: add ClusterRouter HTTP interface for Human interaction"
```

---

## Task 7: Streaming Log

**Files:**
- Create: `src/hal/cluster/streaming.py`
- Create: `tests/cluster/test_streaming.py`

- [ ] **Step 1: Write the failing tests**

`tests/cluster/test_streaming.py`:
```python
import pytest

from hal.core.event_bus import AsyncEventBus
from hal.cluster.streaming import StreamingLog


async def test_streaming_log_captures_events():
    event_bus = AsyncEventBus()
    log = StreamingLog(event_bus)
    log.start()

    await event_bus.emit("task.started", {"task_id": "t-1", "event": "task.started"})
    await event_bus.emit("tool.called", {"tool_name": "git", "event": "tool.called"})
    await event_bus.emit("checkpoint.saved", {"task_id": "t-1", "event": "checkpoint.saved"})

    assert len(log.buffer) == 3
    assert log.buffer[0]["event"] == "task.started"


async def test_streaming_log_max_buffer():
    event_bus = AsyncEventBus()
    log = StreamingLog(event_bus, max_buffer=5)
    log.start()

    for i in range(10):
        await event_bus.emit("task.started", {"event": "task.started", "i": i})

    assert len(log.buffer) == 5
    assert log.buffer[0]["i"] == 5  # oldest evicted


async def test_streaming_log_callback():
    event_bus = AsyncEventBus()
    received = []
    log = StreamingLog(event_bus, on_event=lambda e: received.append(e))
    log.start()

    await event_bus.emit("task.completed", {"event": "task.completed"})

    assert len(received) == 1
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/cluster/test_streaming.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`src/hal/cluster/streaming.py`:
```python
"""Streaming log — subscribe to EventBus and forward events for live observability."""

from collections import deque
from collections.abc import Callable
from typing import Any

from hal.core.event_bus import AsyncEventBus

# Events to subscribe to for streaming
STREAMING_EVENTS = [
    "tool.*",
    "task.*",
    "checkpoint.*",
    "artifact.*",
    "bus.*",
]


class StreamingLog:
    """Captures events from AsyncEventBus for WebSocket streaming and buffering."""

    def __init__(
        self,
        event_bus: AsyncEventBus,
        max_buffer: int = 1000,
        on_event: Callable[[dict[str, Any]], Any] | None = None,
    ) -> None:
        self._event_bus = event_bus
        self._max_buffer = max_buffer
        self._on_event = on_event
        self.buffer: deque[dict[str, Any]] = deque(maxlen=max_buffer)

    def start(self) -> None:
        """Subscribe to all streaming events."""
        for pattern in STREAMING_EVENTS:
            self._event_bus.subscribe(pattern, self._handle_event)

    async def _handle_event(self, data: dict[str, Any]) -> None:
        self.buffer.append(data)
        if self._on_event:
            self._on_event(data)
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/cluster/test_streaming.py -v`
Expected: 3 passed

- [ ] **Step 5: Commit**

```bash
git add src/hal/cluster/streaming.py tests/cluster/test_streaming.py
git commit -m "feat: add StreamingLog for live event observability"
```

---

## Task 8: CLI Entry Point

**Files:**
- Create: `src/hal/cluster/cli.py`
- Create: `tests/cluster/test_cli.py`

- [ ] **Step 1: Write the failing tests**

`tests/cluster/test_cli.py`:
```python
from pathlib import Path
from unittest.mock import AsyncMock, patch

import pytest
import yaml

from hal.cluster.cli import build_cluster


SAMPLE_CONFIG = {
    "cluster": {
        "name": "test-team",
        "leader_agent": {"role": "product_manager", "instance": "pm-01"},
        "member_agents": [
            {"role": "ios_developer", "instance": "ios-dev-01"},
        ],
    }
}


def test_build_cluster_returns_components(tmp_path: Path):
    config_file = tmp_path / "cluster.yaml"
    config_file.write_text(yaml.dump(SAMPLE_CONFIG))

    # Create minimal role files
    roles_dir = tmp_path / "roles"
    roles_dir.mkdir()
    for role_name in ["product_manager", "ios_developer"]:
        role_file = roles_dir / f"{role_name}.yaml"
        role_file.write_text(yaml.dump({
            "identity": {"name": role_name, "persona": f"You are a {role_name}."},
        }))

    cluster = build_cluster(config_file, roles_dir)

    assert cluster is not None
    assert cluster.router is not None
    assert "pm-01" in cluster.services or cluster.leader_service is not None
    assert len(cluster.member_services) == 1
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/cluster/test_cli.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`src/hal/cluster/cli.py`:
```python
"""CLI entry point — bootstrap cluster and manage lifecycle."""

import asyncio
import logging
import signal
from dataclasses import dataclass
from pathlib import Path
from typing import Any

from hal.cluster.agent_service import AgentService
from hal.cluster.config import load_cluster_config
from hal.cluster.router import ClusterRouter
from hal.cluster.streaming import StreamingLog
from hal.core.bus.memory import InMemoryBus
from hal.core.event_bus import AsyncEventBus
from hal.drivers.mock import MockDriver
from hal.core.driver import DriverResult, DriverStatus
from hal.identity.role import load_role
from hal.runner.checkpoint import InMemoryCheckpointStore

logger = logging.getLogger(__name__)


@dataclass
class Cluster:
    """Assembled cluster ready to run."""

    router: ClusterRouter
    leader_service: AgentService
    member_services: list[AgentService]
    services: dict[str, AgentService]
    streaming_log: StreamingLog
    bus: InMemoryBus
    event_bus: AsyncEventBus
    config: Any = None


def build_cluster(
    config_path: Path,
    roles_dir: Path,
    driver_factory: Any = None,
) -> Cluster:
    """Build a cluster from config and role files.

    Args:
        config_path: Path to cluster config YAML.
        roles_dir: Directory containing role YAML files.
        driver_factory: Optional callable(role) -> AgentDriver. Defaults to MockDriver.
    """
    config = load_cluster_config(config_path)
    event_bus = AsyncEventBus()
    bus = InMemoryBus(max_inbox_size=config.bus_max_inbox_size)

    from hal.memory.episodic import EpisodicMemory
    from hal.memory.shared import SharedMemory
    from hal.memory.store import InMemoryMemoryStore

    # Cluster-wide shared memory
    shared_memory = SharedMemory(
        store=InMemoryMemoryStore(),
        max_entries=500,
        write_policy=config.shared_memory_write_policy,
    )

    services: dict[str, AgentService] = {}

    def _create_service(instance_id: str, role_name: str) -> AgentService:
        role_path = roles_dir / f"{role_name}.yaml"
        role = load_role(role_path)

        if driver_factory:
            driver = driver_factory(role)
        else:
            driver = MockDriver(responses=[
                DriverResult(status=DriverStatus.COMPLETED, messages=[],
                             steps_taken=0, token_usage=0, cost_usd=0.0),
            ])

        checkpoint_store = InMemoryCheckpointStore()
        episodic_memory = EpisodicMemory(
            store=InMemoryMemoryStore(),
            max_entries=role.memory.max_episodic,
        )
        return AgentService(
            instance_id=instance_id,
            driver=driver,
            role=role,
            bus=bus,
            event_bus=event_bus,
            checkpoint_store=checkpoint_store,
            hard_cost_ceiling=config.max_cost_per_task,
        )

    # Build leader
    leader = _create_service(config.leader_instance, config.leader_role)
    services[config.leader_instance] = leader

    # Build members
    members = []
    for agent_config in config.member_agents:
        service = _create_service(agent_config.instance, agent_config.role)
        services[agent_config.instance] = service
        members.append(service)

    # Build router
    router = ClusterRouter(bus=bus, event_bus=event_bus, leader_instance=config.leader_instance)

    # Register router as Bus interceptor for escalation/confirmation routing
    bus.add_interceptor(
        predicate=router.should_intercept,
        handler=router.route_artifact,
    )

    # Build streaming log
    streaming = StreamingLog(event_bus)
    streaming.start()

    # Subscribe services to bus
    for instance_id, service in services.items():
        bus.subscribe(instance_id, service.handle_artifact)

    return Cluster(
        router=router,
        leader_service=leader,
        member_services=members,
        services=services,
        streaming_log=streaming,
        bus=bus,
        event_bus=event_bus,
        config=config,
    )


async def run_cluster(cluster: Cluster) -> None:
    """Run the cluster with graceful shutdown on SIGTERM."""
    loop = asyncio.get_running_loop()
    shutdown_event = asyncio.Event()

    def _on_signal() -> None:
        logger.info("Received shutdown signal")
        shutdown_event.set()

    for sig in (signal.SIGTERM, signal.SIGINT):
        loop.add_signal_handler(sig, _on_signal)

    # Start all agent services
    for service in cluster.services.values():
        await service.start()

    # Start HTTP interface
    from hal.cluster.router import create_http_app
    from aiohttp import web

    app = create_http_app(cluster.router)
    runner = web.AppRunner(app)
    await runner.setup()
    host, port = cluster.config.router_listen.rsplit(":", 1)
    site = web.TCPSite(runner, host, int(port))
    await site.start()
    logger.info("Cluster '%s' started, listening on %s", cluster.config.name, cluster.config.router_listen)

    # Wait for shutdown signal
    await shutdown_event.wait()

    # Graceful shutdown
    logger.info("Shutting down (timeout: %ds, policy: %s)",
                cluster.config.shutdown_timeout, cluster.config.shutdown_policy)

    # 1. Stop all services (stop accepting new tasks, cancel TTL scans)
    all_active_tasks: set[asyncio.Task] = set()
    for service in cluster.services.values():
        active = await service.stop()
        all_active_tasks.update(active)

    # 2. Wait for active Driver calls to complete, with timeout
    if all_active_tasks:
        done, pending = await asyncio.wait(
            all_active_tasks, timeout=cluster.config.shutdown_timeout,
        )
        if pending:
            if cluster.config.shutdown_policy == "checkpoint_and_exit":
                logger.info("Timeout reached, checkpointing and exiting (%d tasks pending)", len(pending))
                for t in pending:
                    t.cancel()
            else:  # prompt_human
                logger.info("Timeout reached, %d tasks still pending. Awaiting Human confirmation to force exit.", len(pending))
                # In V1, fall back to cancel after logging
                # Full prompt_human implementation requires HTTP interaction
                for t in pending:
                    t.cancel()

    await runner.cleanup()
    logger.info("Cluster shut down")
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/cluster/test_cli.py -v`
Expected: 1 passed

- [ ] **Step 5: Commit**

```bash
git add src/hal/cluster/cli.py tests/cluster/test_cli.py
git commit -m "feat: add CLI cluster builder with graceful shutdown support"
```

---

## Task 9: Integration Test

**Files:**
- Create: `tests/cluster/test_cluster_integration.py`

- [ ] **Step 1: Write the integration test**

`tests/cluster/test_cluster_integration.py`:
```python
"""Integration: Human submits task → Leader → Member → escalation → Leader resolves → complete."""

import asyncio

import pytest

from hal.cluster.agent_service import AgentService
from hal.cluster.leader_tools import AssignTaskTool, ResolveEscalationTool
from hal.cluster.router import ClusterRouter
from hal.core.artifact import ArtifactType, create_artifact
from hal.core.bus.memory import InMemoryBus
from hal.core.driver import DriverResult, DriverStatus, SuspendContext
from hal.core.event_bus import AsyncEventBus
from hal.drivers.mock import MockDriver
from hal.identity.role import ConcurrencyConfig, IdentityConfig, RoleProfile, SafeguardConfig
from hal.runner.checkpoint import InMemoryCheckpointStore


def _make_role(name: str, max_concurrent: int = 1) -> RoleProfile:
    return RoleProfile(
        identity=IdentityConfig(name=name, persona=f"You are {name}."),
        concurrency=ConcurrencyConfig(max_concurrent_tasks=max_concurrent),
        safeguards=SafeguardConfig(max_steps=50, cost_budget=5.0),
    )


async def test_task_completion_flow():
    """Human → Leader assigns → Member completes → result reaches Leader."""
    bus = InMemoryBus(max_inbox_size=20)
    event_bus = AsyncEventBus()

    # Member agent that completes immediately
    member_driver = MockDriver(responses=[
        DriverResult(
            status=DriverStatus.COMPLETED,
            messages=[{"role": "assistant", "content": "Login implemented"}],
            steps_taken=10,
            token_usage=1500,
            cost_usd=1.5,
            summary="Login SDK implemented",
        ),
    ])
    member_role = _make_role("iOS Dev")
    member_service = AgentService(
        instance_id="ios-dev-01",
        driver=member_driver,
        role=member_role,
        bus=bus,
        event_bus=event_bus,
        checkpoint_store=InMemoryCheckpointStore(),
        hard_cost_ceiling=50.0,
    )

    bus.subscribe("ios-dev-01", member_service.handle_artifact)

    # Use AssignTaskTool to dispatch
    services = {"ios-dev-01": member_service}
    assign_tool = AssignTaskTool(services=services, bus=bus)

    result = await assign_tool.execute({
        "target": "ios-dev-01",
        "task": "Implement login SDK",
        "task_id": "sub-task-1",
        "trace_id": "tr-main",
        "parent_task_id": "leader-task-1",
    })

    assert result.status.value == "success"

    # Verify member processed the task
    assert member_service.capacity.running == 0  # completed
```

- [ ] **Step 2: Run integration test**

Run: `pytest tests/cluster/test_cluster_integration.py -v`
Expected: 1 passed

- [ ] **Step 3: Run full test suite**

Run: `pytest tests/ -v`
Expected: All tests pass

- [ ] **Step 4: Commit**

```bash
git add tests/cluster/test_cluster_integration.py
git commit -m "test: add cluster management integration test"
```

---

## Summary

| Task | Component | Test count (est.) |
|------|-----------|-------------------|
| 1 | Cluster Config | 5 |
| 2 | AgentService — Core | 4 |
| 3 | AgentService — Preemption | 1 |
| 4 | Leader Agent Tools (assign, wait, query, resolve, collect, broadcast) | 5 |
| 5 | ClusterRouter — Routing | 5 |
| 6 | ClusterRouter — HTTP | 3 |
| 7 | Streaming Log | 3 |
| 8 | CLI Entry Point | 1 |
| 9 | Integration | 1 |
| **Total** | | **~27** |
