# Plan 4: Task Runner + MockDriver — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build Layer 4 — TaskRunner orchestrator, Working Memory, Checkpoint (with SQLite backend), Safeguards, resume flow, and a MockDriver for testing all flows.

**Architecture:** TaskRunner is a thin orchestrator: prepare context → call Driver → post-execution (memory, checkpoint, safeguards) → handle result status. Working Memory holds opaque conversation history. Checkpoint persists full task state for suspend/resume. Safeguards track cumulative steps/cost. MockDriver implements the AgentDriver protocol with configurable behaviors for testing.

**Tech Stack:** Python 3.9+, asyncio, sqlite3 (stdlib), pytest, pytest-asyncio

**Spec:** `docs/superpowers/specs/2026-04-27-hal-design.md` — Layer 4: Task Runner

**Depends on:** Plan 1 (Bus, Artifacts, EventBus, Driver protocol), Plan 2 (SandboxedTool), Plan 3 (RoleProfile, PromptBuilder)

---

## File Structure

```
src/hal/runner/
├── __init__.py
├── working_memory.py    # Opaque conversation history management
├── safeguards.py        # Cumulative step/cost tracking, budget enforcement
├── checkpoint.py        # Checkpoint model + InMemory/SQLite stores
└── task_runner.py       # TaskRunner orchestrator (run/resume flows)

src/hal/drivers/
├── __init__.py
└── mock.py              # MockDriver for testing

tests/runner/
├── __init__.py
├── test_working_memory.py
├── test_safeguards.py
├── test_checkpoint.py
├── test_task_runner.py
└── test_runner_integration.py

tests/drivers/
├── __init__.py
└── test_mock_driver.py
```

---

## Task 1: Working Memory

**Files:**
- Create: `src/hal/runner/__init__.py`
- Create: `src/hal/runner/working_memory.py`
- Create: `tests/runner/__init__.py`
- Create: `tests/runner/test_working_memory.py`

- [ ] **Step 1: Write the failing tests**

`tests/runner/test_working_memory.py`:
```python
from hal.runner.working_memory import WorkingMemory


def test_append_and_get_messages():
    wm = WorkingMemory()
    wm.append({"role": "user", "content": "Hello"})
    wm.append({"role": "assistant", "content": "Hi"})

    messages = wm.get_messages()
    assert len(messages) == 2
    assert messages[0]["role"] == "user"
    assert messages[1]["role"] == "assistant"


def test_replace_messages():
    wm = WorkingMemory()
    wm.append({"role": "user", "content": "old"})

    new_messages = [
        {"role": "user", "content": "new"},
        {"role": "assistant", "content": "response"},
    ]
    wm.replace(new_messages)

    assert wm.get_messages() == new_messages


def test_snapshot_and_restore():
    wm = WorkingMemory()
    wm.append({"role": "user", "content": "Hello"})
    wm.append({"role": "assistant", "content": "Hi"})

    snapshot = wm.snapshot()

    # Mutate the working memory
    wm.append({"role": "user", "content": "More"})
    assert len(wm.get_messages()) == 3

    # Restore from snapshot
    wm.restore(snapshot)
    assert len(wm.get_messages()) == 2
    assert wm.get_messages()[-1]["content"] == "Hi"


def test_snapshot_is_independent_copy():
    wm = WorkingMemory()
    wm.append({"role": "user", "content": "Hello"})

    snapshot = wm.snapshot()
    wm.append({"role": "user", "content": "World"})

    # Snapshot should not be affected
    restored = WorkingMemory()
    restored.restore(snapshot)
    assert len(restored.get_messages()) == 1


def test_clear():
    wm = WorkingMemory()
    wm.append({"role": "user", "content": "Hello"})
    wm.clear()
    assert wm.get_messages() == []


def test_messages_are_opaque():
    """Working Memory stores any dict structure without parsing."""
    wm = WorkingMemory()
    wm.append({"role": "user", "content": [{"type": "text", "text": "multimodal"}]})
    msg = wm.get_messages()[0]
    assert msg["content"][0]["type"] == "text"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/runner/test_working_memory.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`src/hal/runner/__init__.py`:
```python
"""HAL Task Runner — thin orchestrator delegating execution to AgentDriver."""
```

`src/hal/runner/working_memory.py`:
```python
"""Working Memory — opaque conversation history for a single task.

HAL stores and passes messages without parsing their internal structure.
The format is determined by the Driver.
"""

import copy
from typing import Any


class WorkingMemory:
    """Per-task conversation history. Opaque to HAL."""

    def __init__(self) -> None:
        self._messages: list[dict[str, Any]] = []

    def append(self, message: dict[str, Any]) -> None:
        """Append a message to the conversation history."""
        self._messages.append(message)

    def get_messages(self) -> list[dict[str, Any]]:
        """Get all messages (returns a copy)."""
        return list(self._messages)

    def replace(self, messages: list[dict[str, Any]]) -> None:
        """Replace all messages with Driver's updated version."""
        self._messages = list(messages)

    def snapshot(self) -> list[dict[str, Any]]:
        """Create an independent deep copy for checkpointing."""
        return copy.deepcopy(self._messages)

    def restore(self, snapshot: list[dict[str, Any]]) -> None:
        """Restore from a checkpoint snapshot."""
        self._messages = copy.deepcopy(snapshot)

    def clear(self) -> None:
        """Discard all messages."""
        self._messages.clear()
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/runner/test_working_memory.py -v`
Expected: 6 passed

- [ ] **Step 5: Commit**

```bash
git add src/hal/runner/ tests/runner/
git commit -m "feat: add WorkingMemory for opaque conversation history"
```

---

## Task 2: MockDriver

**Files:**
- Create: `src/hal/drivers/__init__.py`
- Create: `src/hal/drivers/mock.py`
- Create: `tests/drivers/__init__.py`
- Create: `tests/drivers/test_mock_driver.py`

- [ ] **Step 1: Write the failing tests**

`tests/drivers/test_mock_driver.py`:
```python
import pytest

from hal.core.driver import (
    CancellationToken,
    DriverResult,
    DriverStatus,
    Lesson,
    LessonScope,
    SuspendContext,
)
from hal.drivers.mock import MockDriver


async def test_mock_driver_completed():
    driver = MockDriver(responses=[
        DriverResult(
            status=DriverStatus.COMPLETED,
            messages=[{"role": "assistant", "content": "Done"}],
            steps_taken=3,
            token_usage=500,
            cost_usd=0.5,
            summary="Task completed",
        ),
    ])

    result = await driver.run(
        messages=[{"role": "user", "content": "Do something"}],
        tools=[],
        system_prompt="You are a test agent.",
        max_steps=50,
        cost_budget=5.0,
    )

    assert result.status == DriverStatus.COMPLETED
    assert result.steps_taken == 3


async def test_mock_driver_escalated():
    driver = MockDriver(responses=[
        DriverResult(
            status=DriverStatus.ESCALATED,
            messages=[],
            steps_taken=2,
            token_usage=300,
            cost_usd=0.3,
            escalation_reason="Need help",
        ),
    ])

    result = await driver.run(
        messages=[], tools=[], system_prompt="", max_steps=50, cost_budget=5.0,
    )
    assert result.status == DriverStatus.ESCALATED
    assert result.escalation_reason == "Need help"


async def test_mock_driver_suspended_with_context():
    driver = MockDriver(responses=[
        DriverResult(
            status=DriverStatus.SUSPENDED,
            messages=[],
            steps_taken=1,
            token_usage=100,
            cost_usd=0.1,
            suspend_context=SuspendContext(resume_event_id="evt-123"),
        ),
    ])

    result = await driver.run(
        messages=[], tools=[], system_prompt="", max_steps=50, cost_budget=5.0,
    )
    assert result.status == DriverStatus.SUSPENDED
    assert result.suspend_context.resume_event_id == "evt-123"


async def test_mock_driver_sequential_responses():
    """Multiple calls return responses in order."""
    driver = MockDriver(responses=[
        DriverResult(
            status=DriverStatus.SUSPENDED, messages=[], steps_taken=1, token_usage=100, cost_usd=0.1,
            cost_usd=0.1,
            suspend_context=SuspendContext(resume_event_id="evt-1"),
        ),
        DriverResult(
            status=DriverStatus.COMPLETED, messages=[], steps_taken=2, token_usage=200, cost_usd=0.2,
            summary="Resumed and completed",
        ),
    ])

    r1 = await driver.run(
        messages=[], tools=[], system_prompt="", max_steps=50, cost_budget=5.0,
    )
    assert r1.status == DriverStatus.SUSPENDED

    r2 = await driver.run(
        messages=[], tools=[], system_prompt="", max_steps=50, cost_budget=5.0,
    )
    assert r2.status == DriverStatus.COMPLETED


async def test_mock_driver_with_lessons():
    lesson = Lesson(
        content="Use pinned deps",
        trigger="build_failure",
        confidence=0.7,
        tags=["deps"],
        source_task="task-1",
    )
    driver = MockDriver(responses=[
        DriverResult(
            status=DriverStatus.COMPLETED, messages=[], steps_taken=5,
            token_usage=800, cost_usd=0.8, lessons=[lesson],
        ),
    ])

    result = await driver.run(
        messages=[], tools=[], system_prompt="", max_steps=50, cost_budget=5.0,
    )
    assert len(result.lessons) == 1
    assert result.lessons[0].content == "Use pinned deps"


async def test_mock_driver_respects_cancellation():
    driver = MockDriver(
        responses=[
            DriverResult(
                status=DriverStatus.COMPLETED, messages=[], steps_taken=10,
                token_usage=1000, cost_usd=1.0,
            ),
        ],
        support_cancellation=True,
    )

    token = CancellationToken()
    token.cancel()

    result = await driver.run(
        messages=[], tools=[], system_prompt="", max_steps=50, cost_budget=5.0,
        cancellation_token=token,
    )
    assert result.status == DriverStatus.SUSPENDED


async def test_mock_driver_records_calls():
    driver = MockDriver(responses=[
        DriverResult(status=DriverStatus.COMPLETED, messages=[], steps_taken=1, token_usage=100, cost_usd=0.1),
    ])

    await driver.run(
        messages=[{"role": "user", "content": "Hi"}],
        tools=["tool_a"],
        system_prompt="prompt",
        max_steps=30,
        cost_budget=3.0,
    )

    assert len(driver.calls) == 1
    assert driver.calls[0]["system_prompt"] == "prompt"
    assert driver.calls[0]["max_steps"] == 30
    assert driver.calls[0]["messages"] == [{"role": "user", "content": "Hi"}]
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/drivers/test_mock_driver.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`src/hal/drivers/__init__.py`:
```python
"""HAL AgentDriver implementations."""
```

`src/hal/drivers/mock.py`:
```python
"""MockDriver — configurable AgentDriver for testing.

Supports scripted response sequences, cancellation simulation, and call recording.
"""

from typing import Any

from hal.core.driver import (
    CancellationToken,
    DriverResult,
    DriverStatus,
    SuspendContext,
)


class MockDriver:
    """Mock AgentDriver that returns pre-configured responses in sequence.

    Args:
        responses: Ordered list of DriverResults to return on each run() call.
        support_cancellation: If True, checks cancellation_token and returns
            SUSPENDED when cancelled (before returning the scripted response).
    """

    def __init__(
        self,
        responses: list[DriverResult],
        support_cancellation: bool = False,
    ) -> None:
        self._responses = list(responses)
        self._index = 0
        self._support_cancellation = support_cancellation
        self.calls: list[dict[str, Any]] = []

    async def run(
        self,
        *,
        messages: list[dict[str, Any]],
        tools: list[Any],
        system_prompt: str,
        max_steps: int,
        cost_budget: float,
        cancellation_token: CancellationToken | None = None,
    ) -> DriverResult:
        # Record call
        self.calls.append({
            "messages": messages,
            "tools": tools,
            "system_prompt": system_prompt,
            "max_steps": max_steps,
            "cost_budget": cost_budget,
        })

        # Check cancellation
        if (
            self._support_cancellation
            and cancellation_token is not None
            and cancellation_token.is_cancelled
        ):
            return DriverResult(
                status=DriverStatus.SUSPENDED,
                messages=messages,
                steps_taken=0,
                token_usage=0,
                cost_usd=0.0,
                suspend_context=SuspendContext(),
            )

        # Return scripted response
        if self._index < len(self._responses):
            result = self._responses[self._index]
            self._index += 1
            return result

        # Fallback if exhausted
        return DriverResult(
            status=DriverStatus.FAILED,
            messages=messages,
            steps_taken=0,
            token_usage=0, cost_usd=0.0,
            summary="MockDriver: no more scripted responses",
        )
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/drivers/test_mock_driver.py -v`
Expected: 7 passed

- [ ] **Step 5: Commit**

```bash
git add src/hal/drivers/ tests/drivers/
git commit -m "feat: add MockDriver with scripted responses and call recording"
```

---

## Task 3: Safeguards

**Files:**
- Create: `src/hal/runner/safeguards.py`
- Create: `tests/runner/test_safeguards.py`

- [ ] **Step 1: Write the failing tests**

`tests/runner/test_safeguards.py`:
```python
import pytest

from hal.runner.safeguards import Safeguards


def test_initial_state():
    sg = Safeguards(max_steps=50, cost_budget=5.0, max_escalations=3, hard_cost_ceiling=50.0)
    assert sg.steps_used == 0
    assert sg.cost_used == 0.0
    assert sg.escalations_used == 0
    assert sg.remaining_steps == 50
    assert sg.remaining_budget == 5.0


def test_record_usage():
    sg = Safeguards(max_steps=50, cost_budget=5.0, max_escalations=3, hard_cost_ceiling=50.0)
    sg.record_usage(steps=10, cost=1.5)
    assert sg.steps_used == 10
    assert sg.cost_used == 1.5
    assert sg.remaining_steps == 40
    assert sg.remaining_budget == 3.5


def test_record_usage_cumulative():
    sg = Safeguards(max_steps=50, cost_budget=5.0, max_escalations=3, hard_cost_ceiling=50.0)
    sg.record_usage(steps=10, cost=1.0)
    sg.record_usage(steps=5, cost=0.5)
    assert sg.steps_used == 15
    assert sg.cost_used == 1.5


def test_record_escalation():
    sg = Safeguards(max_steps=50, cost_budget=5.0, max_escalations=3, hard_cost_ceiling=50.0)
    assert sg.can_escalate is True
    sg.record_escalation()
    sg.record_escalation()
    sg.record_escalation()
    assert sg.can_escalate is False
    assert sg.escalations_used == 3


def test_hard_cost_ceiling_exceeded():
    sg = Safeguards(max_steps=50, cost_budget=5.0, max_escalations=3, hard_cost_ceiling=10.0)
    sg.record_usage(steps=5, cost=9.0)
    assert sg.hard_ceiling_exceeded is False
    sg.record_usage(steps=1, cost=2.0)
    assert sg.hard_ceiling_exceeded is True


def test_snapshot_and_restore():
    sg = Safeguards(max_steps=50, cost_budget=5.0, max_escalations=3, hard_cost_ceiling=50.0)
    sg.record_usage(steps=10, cost=1.5)
    sg.record_escalation()

    snapshot = sg.snapshot()

    sg.record_usage(steps=20, cost=2.0)
    sg.record_escalation()

    sg.restore(snapshot)
    assert sg.steps_used == 10
    assert sg.cost_used == 1.5
    assert sg.escalations_used == 1
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/runner/test_safeguards.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`src/hal/runner/safeguards.py`:
```python
"""Safeguards — cumulative execution limit tracking across Driver calls.

Tracks max steps, cost budget, escalation count, and hard cost ceiling.
Remaining steps/budget are passed to the Driver as constraints.
"""

from dataclasses import dataclass
from typing import Any


class Safeguards:
    """Track cumulative execution limits across suspend/resume cycles."""

    def __init__(
        self,
        max_steps: int,
        cost_budget: float,
        max_escalations: int,
        hard_cost_ceiling: float,
    ) -> None:
        self._max_steps = max_steps
        self._cost_budget = cost_budget
        self._max_escalations = max_escalations
        self._hard_cost_ceiling = hard_cost_ceiling
        self._steps_used: int = 0
        self._cost_used: float = 0.0
        self._escalations_used: int = 0

    @property
    def steps_used(self) -> int:
        return self._steps_used

    @property
    def cost_used(self) -> float:
        return self._cost_used

    @property
    def escalations_used(self) -> int:
        return self._escalations_used

    @property
    def remaining_steps(self) -> int:
        return max(0, self._max_steps - self._steps_used)

    @property
    def remaining_budget(self) -> float:
        return max(0.0, self._cost_budget - self._cost_used)

    @property
    def can_escalate(self) -> bool:
        return self._escalations_used < self._max_escalations

    @property
    def hard_ceiling_exceeded(self) -> bool:
        return self._cost_used > self._hard_cost_ceiling

    def record_usage(self, *, steps: int, cost: float) -> None:
        """Record Driver-reported usage from a single run."""
        self._steps_used += steps
        self._cost_used += cost

    def record_escalation(self) -> None:
        """Record one escalation event."""
        self._escalations_used += 1

    def snapshot(self) -> dict[str, Any]:
        """Snapshot for checkpoint."""
        return {
            "steps_used": self._steps_used,
            "cost_used": self._cost_used,
            "escalations_used": self._escalations_used,
        }

    def restore(self, data: dict[str, Any]) -> None:
        """Restore from checkpoint snapshot."""
        self._steps_used = data["steps_used"]
        self._cost_used = data["cost_used"]
        self._escalations_used = data["escalations_used"]
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/runner/test_safeguards.py -v`
Expected: 6 passed

- [ ] **Step 5: Commit**

```bash
git add src/hal/runner/safeguards.py tests/runner/test_safeguards.py
git commit -m "feat: add Safeguards for cumulative execution limit tracking"
```

---

## Task 4: Checkpoint (InMemory + SQLite)

**Files:**
- Create: `src/hal/runner/checkpoint.py`
- Create: `tests/runner/test_checkpoint.py`

- [ ] **Step 1: Write the failing tests**

`tests/runner/test_checkpoint.py`:
```python
import pytest

from hal.runner.checkpoint import (
    SCHEMA_VERSION,
    Checkpoint,
    InMemoryCheckpointStore,
    SQLiteCheckpointStore,
)


def _make_checkpoint(
    task_id: str = "task-1",
    agent_id: str = "ios-dev-01",
    status: str = "suspended",
) -> Checkpoint:
    return Checkpoint(
        schema_version=SCHEMA_VERSION,
        task_id=task_id,
        agent_id=agent_id,
        driver_type="MockDriver",
        messages=[{"role": "user", "content": "hello"}],
        safeguards_state={"steps_used": 5, "cost_used": 1.0, "escalations_used": 0},
        task_status=status,
        suspended_at="2026-04-29T10:00:00Z",
        context_data={"escalation_reason": "need help"},
    )


class TestInMemoryCheckpointStore:
    @pytest.fixture
    def store(self) -> InMemoryCheckpointStore:
        return InMemoryCheckpointStore()

    def test_save_and_load(self, store: InMemoryCheckpointStore):
        cp = _make_checkpoint()
        store.save(cp)

        loaded = store.load("task-1")
        assert loaded is not None
        assert loaded.task_id == "task-1"
        assert loaded.agent_id == "ios-dev-01"
        assert loaded.messages == [{"role": "user", "content": "hello"}]

    def test_load_nonexistent(self, store: InMemoryCheckpointStore):
        assert store.load("nonexistent") is None

    def test_overwrite(self, store: InMemoryCheckpointStore):
        store.save(_make_checkpoint(status="suspended"))
        store.save(_make_checkpoint(status="escalated"))

        loaded = store.load("task-1")
        assert loaded.task_status == "escalated"

    def test_delete(self, store: InMemoryCheckpointStore):
        store.save(_make_checkpoint())
        store.delete("task-1")
        assert store.load("task-1") is None

    def test_list_by_agent(self, store: InMemoryCheckpointStore):
        store.save(_make_checkpoint(task_id="t-1", agent_id="a"))
        store.save(_make_checkpoint(task_id="t-2", agent_id="a"))
        store.save(_make_checkpoint(task_id="t-3", agent_id="b"))

        result = store.list_by_agent("a")
        assert sorted(cp.task_id for cp in result) == ["t-1", "t-2"]

    def test_schema_version_mismatch(self, store: InMemoryCheckpointStore):
        cp = _make_checkpoint()
        store.save(cp)

        # Tamper with stored checkpoint's schema version
        stored = store.load("task-1")
        tampered = Checkpoint(
            schema_version="0.0.0-invalid",
            task_id=stored.task_id,
            agent_id=stored.agent_id,
            driver_type=stored.driver_type,
            messages=stored.messages,
            safeguards_state=stored.safeguards_state,
            task_status=stored.task_status,
            suspended_at=stored.suspended_at,
            context_data=stored.context_data,
        )
        store.save(tampered)

        with pytest.raises(ValueError, match="schema_version"):
            store.load_validated("task-1")


class TestSQLiteCheckpointStore:
    @pytest.fixture
    def store(self, tmp_path) -> SQLiteCheckpointStore:
        return SQLiteCheckpointStore(tmp_path / "checkpoints.db")

    def test_save_and_load(self, store: SQLiteCheckpointStore):
        cp = _make_checkpoint()
        store.save(cp)

        loaded = store.load("task-1")
        assert loaded is not None
        assert loaded.task_id == "task-1"
        assert loaded.messages == [{"role": "user", "content": "hello"}]

    def test_load_nonexistent(self, store: SQLiteCheckpointStore):
        assert store.load("nonexistent") is None

    def test_overwrite(self, store: SQLiteCheckpointStore):
        store.save(_make_checkpoint(status="suspended"))
        store.save(_make_checkpoint(status="escalated"))

        loaded = store.load("task-1")
        assert loaded.task_status == "escalated"

    def test_delete(self, store: SQLiteCheckpointStore):
        store.save(_make_checkpoint())
        store.delete("task-1")
        assert store.load("task-1") is None

    def test_list_by_agent(self, store: SQLiteCheckpointStore):
        store.save(_make_checkpoint(task_id="t-1", agent_id="a"))
        store.save(_make_checkpoint(task_id="t-2", agent_id="a"))
        store.save(_make_checkpoint(task_id="t-3", agent_id="b"))

        result = store.list_by_agent("a")
        assert sorted(cp.task_id for cp in result) == ["t-1", "t-2"]
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/runner/test_checkpoint.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`src/hal/runner/checkpoint.py`:
```python
"""Checkpoint — task state persistence for suspend/resume.

Includes all state needed for recovery: Working Memory snapshot,
Safeguards state, task status, and context-specific data.
"""

import json
import sqlite3
from dataclasses import dataclass, field
from pathlib import Path
from typing import Any, Protocol

SCHEMA_VERSION = "0.1.0"


@dataclass
class Checkpoint:
    """Full task state snapshot for suspend/resume."""

    schema_version: str
    task_id: str
    agent_id: str
    driver_type: str
    messages: list[dict[str, Any]]
    safeguards_state: dict[str, Any]
    task_status: str
    suspended_at: str | None = None
    context_data: dict[str, Any] = field(default_factory=dict)


class CheckpointStore(Protocol):
    """Protocol for checkpoint persistence backends."""

    def save(self, checkpoint: Checkpoint) -> None: ...
    def load(self, task_id: str) -> Checkpoint | None: ...
    def delete(self, task_id: str) -> None: ...
    def list_by_agent(self, agent_id: str) -> list[Checkpoint]: ...


class InMemoryCheckpointStore:
    """In-memory checkpoint store for testing."""

    def __init__(self) -> None:
        self._store: dict[str, Checkpoint] = {}

    def save(self, checkpoint: Checkpoint) -> None:
        self._store[checkpoint.task_id] = checkpoint

    def load(self, task_id: str) -> Checkpoint | None:
        return self._store.get(task_id)

    def load_validated(self, task_id: str) -> Checkpoint | None:
        """Load with schema version validation."""
        cp = self.load(task_id)
        if cp is not None and cp.schema_version != SCHEMA_VERSION:
            raise ValueError(
                f"Checkpoint schema_version mismatch: "
                f"expected '{SCHEMA_VERSION}', got '{cp.schema_version}'"
            )
        return cp

    def delete(self, task_id: str) -> None:
        self._store.pop(task_id, None)

    def list_by_agent(self, agent_id: str) -> list[Checkpoint]:
        return [cp for cp in self._store.values() if cp.agent_id == agent_id]


class SQLiteCheckpointStore:
    """SQLite-backed checkpoint store for production."""

    def __init__(self, db_path: Path) -> None:
        self._conn = sqlite3.connect(str(db_path))
        self._conn.execute("PRAGMA journal_mode=WAL")
        self._conn.execute("""
            CREATE TABLE IF NOT EXISTS checkpoints (
                task_id TEXT PRIMARY KEY,
                agent_id TEXT NOT NULL,
                data TEXT NOT NULL
            )
        """)
        self._conn.commit()

    def save(self, checkpoint: Checkpoint) -> None:
        data = json.dumps({
            "schema_version": checkpoint.schema_version,
            "task_id": checkpoint.task_id,
            "agent_id": checkpoint.agent_id,
            "driver_type": checkpoint.driver_type,
            "messages": checkpoint.messages,
            "safeguards_state": checkpoint.safeguards_state,
            "task_status": checkpoint.task_status,
            "suspended_at": checkpoint.suspended_at,
            "context_data": checkpoint.context_data,
        })
        self._conn.execute(
            "INSERT OR REPLACE INTO checkpoints (task_id, agent_id, data) VALUES (?, ?, ?)",
            (checkpoint.task_id, checkpoint.agent_id, data),
        )
        self._conn.commit()

    def load(self, task_id: str) -> Checkpoint | None:
        row = self._conn.execute(
            "SELECT data FROM checkpoints WHERE task_id = ?", (task_id,)
        ).fetchone()
        if row is None:
            return None
        return self._deserialize(row[0])

    def load_validated(self, task_id: str) -> Checkpoint | None:
        cp = self.load(task_id)
        if cp is not None and cp.schema_version != SCHEMA_VERSION:
            raise ValueError(
                f"Checkpoint schema_version mismatch: "
                f"expected '{SCHEMA_VERSION}', got '{cp.schema_version}'"
            )
        return cp

    def delete(self, task_id: str) -> None:
        self._conn.execute("DELETE FROM checkpoints WHERE task_id = ?", (task_id,))
        self._conn.commit()

    def list_by_agent(self, agent_id: str) -> list[Checkpoint]:
        rows = self._conn.execute(
            "SELECT data FROM checkpoints WHERE agent_id = ?", (agent_id,)
        ).fetchall()
        return [self._deserialize(row[0]) for row in rows]

    @staticmethod
    def _deserialize(data_str: str) -> Checkpoint:
        d = json.loads(data_str)
        return Checkpoint(
            schema_version=d["schema_version"],
            task_id=d["task_id"],
            agent_id=d["agent_id"],
            driver_type=d["driver_type"],
            messages=d["messages"],
            safeguards_state=d["safeguards_state"],
            task_status=d["task_status"],
            suspended_at=d.get("suspended_at"),
            context_data=d.get("context_data", {}),
        )
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/runner/test_checkpoint.py -v`
Expected: 11 passed

- [ ] **Step 5: Commit**

```bash
git add src/hal/runner/checkpoint.py tests/runner/test_checkpoint.py
git commit -m "feat: add Checkpoint model with InMemory and SQLite stores"
```

---

## Task 5: TaskRunner — Run Flow

**Files:**
- Create: `src/hal/runner/task_runner.py`
- Create: `tests/runner/test_task_runner.py`

- [ ] **Step 1: Write the failing tests**

`tests/runner/test_task_runner.py`:
```python
import pytest

from hal.core.driver import DriverResult, DriverStatus
from hal.core.event_bus import AsyncEventBus
from hal.drivers.mock import MockDriver
from hal.identity.role import (
    IdentityConfig,
    RoleProfile,
    SafeguardConfig,
)
from hal.identity.prompt_builder import PromptBuilder
from hal.runner.checkpoint import InMemoryCheckpointStore
from hal.runner.task_runner import TaskRunner, TaskResult


def _make_role(max_steps: int = 50, cost_budget: float = 5.0) -> RoleProfile:
    return RoleProfile(
        identity=IdentityConfig(name="TestAgent", persona="You are a test agent."),
        safeguards=SafeguardConfig(max_steps=max_steps, cost_budget=cost_budget),
    )


async def test_run_completed():
    driver = MockDriver(responses=[
        DriverResult(
            status=DriverStatus.COMPLETED,
            messages=[{"role": "assistant", "content": "Done"}],
            steps_taken=3,
            token_usage=500,
            cost_usd=0.5,
            summary="Task completed",
        ),
    ])
    role = _make_role()
    event_bus = AsyncEventBus()
    checkpoint_store = InMemoryCheckpointStore()

    runner = TaskRunner(
        agent_id="agent-1",
        driver=driver,
        role=role,
        event_bus=event_bus,
        checkpoint_store=checkpoint_store,
        hard_cost_ceiling=50.0,
    )

    result = await runner.run(task_id="t-1", task_description="Do something", trace_id="tr-1")

    assert result.status == DriverStatus.COMPLETED
    assert result.summary == "Task completed"


async def test_run_failed():
    driver = MockDriver(responses=[
        DriverResult(
            status=DriverStatus.FAILED,
            messages=[],
            steps_taken=1,
            token_usage=100,
            cost_usd=0.1,
            summary="Crashed",
        ),
    ])
    role = _make_role()
    event_bus = AsyncEventBus()
    checkpoint_store = InMemoryCheckpointStore()

    runner = TaskRunner(
        agent_id="agent-1",
        driver=driver,
        role=role,
        event_bus=event_bus,
        checkpoint_store=checkpoint_store,
        hard_cost_ceiling=50.0,
    )

    result = await runner.run(task_id="t-2", task_description="Fail", trace_id="tr-2")
    assert result.status == DriverStatus.FAILED


async def test_run_records_safeguards():
    driver = MockDriver(responses=[
        DriverResult(
            status=DriverStatus.COMPLETED,
            messages=[],
            steps_taken=10,
            token_usage=2000, cost_usd=2.0,
        ),
    ])
    role = _make_role(max_steps=50, cost_budget=5.0)
    event_bus = AsyncEventBus()
    checkpoint_store = InMemoryCheckpointStore()

    runner = TaskRunner(
        agent_id="agent-1",
        driver=driver,
        role=role,
        event_bus=event_bus,
        checkpoint_store=checkpoint_store,
        hard_cost_ceiling=50.0,
    )

    result = await runner.run(task_id="t-3", task_description="Count", trace_id="tr-3")
    assert result.steps_taken == 10


async def test_run_emits_lifecycle_events():
    driver = MockDriver(responses=[
        DriverResult(
            status=DriverStatus.COMPLETED, messages=[], steps_taken=1, token_usage=100, cost_usd=0.1,
        ),
    ])
    role = _make_role()
    event_bus = AsyncEventBus()
    events: list[dict] = []

    async def handler(data: dict) -> None:
        events.append(data)

    event_bus.subscribe("task.*", handler)

    runner = TaskRunner(
        agent_id="agent-1",
        driver=driver,
        role=role,
        event_bus=event_bus,
        checkpoint_store=InMemoryCheckpointStore(),
        hard_cost_ceiling=50.0,
    )

    await runner.run(task_id="t-4", task_description="Events", trace_id="tr-4")

    event_types = [e["event"] for e in events]
    assert "task.started" in event_types
    assert "task.completed" in event_types
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/runner/test_task_runner.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`src/hal/runner/task_runner.py`:
```python
"""TaskRunner — thin orchestrator delegating execution to AgentDriver.

Run flow: prepare context → call Driver → post-execution → handle result status.
Resume flow: restore checkpoint → inject event data → call Driver → handle result.
"""

import logging
from collections.abc import Callable
from dataclasses import dataclass, field
from datetime import datetime, timezone
from typing import Any

from hal.core.artifact import ArtifactType, create_artifact
from hal.core.bus.protocol import Bus
from hal.core.driver import (
    AgentDriver,
    CancellationToken,
    DriverResult,
    DriverStatus,
)
from hal.core.event_bus import AsyncEventBus
from hal.identity.prompt_builder import PromptBuilder
from hal.identity.role import RoleProfile
from hal.runner.checkpoint import (
    SCHEMA_VERSION,
    Checkpoint,
    CheckpointStore,
)
from hal.runner.safeguards import Safeguards
from hal.runner.working_memory import WorkingMemory

logger = logging.getLogger(__name__)


@dataclass
class TaskResult:
    """Result of a TaskRunner.run() or TaskRunner.resume() call."""

    status: DriverStatus
    task_id: str
    summary: str | None = None
    escalation_reason: str | None = None
    steps_taken: int = 0
    lessons: list[Any] = field(default_factory=list)
    output_artifacts: list[dict[str, Any]] = field(default_factory=list)


class TaskRunner:
    """Orchestrates a single task execution lifecycle."""

    def __init__(
        self,
        agent_id: str,
        driver: AgentDriver,
        role: RoleProfile,
        event_bus: AsyncEventBus,
        checkpoint_store: CheckpointStore,
        hard_cost_ceiling: float,
        bus: Bus | None = None,
        cancellation_token: CancellationToken | None = None,
        on_lessons: Callable[[list[Any], str], None] | None = None,
    ) -> None:
        self._agent_id = agent_id
        self._driver = driver
        self._role = role
        self._event_bus = event_bus
        self._checkpoint_store = checkpoint_store
        self._hard_cost_ceiling = hard_cost_ceiling
        self._bus = bus
        self._cancellation_token = cancellation_token
        self._prompt_builder = PromptBuilder(role)
        self._on_lessons = on_lessons  # callback(lessons, task_id) for memory integration

    async def run(
        self,
        task_id: str,
        task_description: str,
        trace_id: str,
        tools: list[Any] | None = None,
        memories: list[str] | None = None,
        parent_task_id: str | None = None,
        concurrent_context: list[str] | None = None,
    ) -> TaskResult:
        """Execute a new task from scratch."""
        wm = WorkingMemory()
        wm.append({"role": "user", "content": task_description})

        if concurrent_context:
            context_text = "\n".join(f"- {ctx}" for ctx in concurrent_context)
            wm.append({"role": "user", "content": f"[CONCURRENT TASKS] Currently running on this agent:\n{context_text}"})

        safeguards = Safeguards(
            max_steps=self._role.safeguards.max_steps,
            cost_budget=self._role.safeguards.cost_budget,
            max_escalations=self._role.safeguards.max_escalations_per_task,
            hard_cost_ceiling=self._hard_cost_ceiling,
        )

        await self._emit_event("task.started", task_id=task_id, trace_id=trace_id)

        return await self._execute(
            task_id=task_id,
            trace_id=trace_id,
            wm=wm,
            safeguards=safeguards,
            tools=tools or [],
            memories=memories,
            parent_task_id=parent_task_id,
        )

    async def resume(
        self,
        task_id: str,
        trace_id: str,
        event_data: dict[str, Any],
        tools: list[Any] | None = None,
        memories: list[str] | None = None,
    ) -> TaskResult:
        """Resume a suspended task from checkpoint."""
        cp = self._checkpoint_store.load_validated(task_id)
        if cp is None:
            return TaskResult(
                status=DriverStatus.FAILED,
                task_id=task_id,
                summary=f"No checkpoint found for task {task_id}",
            )

        wm = WorkingMemory()
        wm.restore(cp.messages)

        # Inject event data into conversation
        wm.append({"role": "user", "content": f"[RESUME EVENT] {event_data}"})

        safeguards = Safeguards(
            max_steps=self._role.safeguards.max_steps,
            cost_budget=self._role.safeguards.cost_budget,
            max_escalations=self._role.safeguards.max_escalations_per_task,
            hard_cost_ceiling=self._hard_cost_ceiling,
        )
        safeguards.restore(cp.safeguards_state)

        await self._emit_event("task.resumed", task_id=task_id, trace_id=trace_id)

        parent_task_id = cp.context_data.get("parent_task_id")

        return await self._execute(
            task_id=task_id,
            trace_id=trace_id,
            wm=wm,
            safeguards=safeguards,
            tools=tools or [],
            memories=memories,
            parent_task_id=parent_task_id,
        )

    async def _execute(
        self,
        task_id: str,
        trace_id: str,
        wm: WorkingMemory,
        safeguards: Safeguards,
        tools: list[Any],
        memories: list[str] | None,
        parent_task_id: str | None = None,
    ) -> TaskResult:
        """Core execution: call Driver → post-execution → handle result."""
        system_prompt = self._prompt_builder.build(memories=memories)

        # Call Driver
        driver_result = await self._driver.run(
            messages=wm.get_messages(),
            tools=tools,
            system_prompt=system_prompt,
            max_steps=safeguards.remaining_steps,
            cost_budget=safeguards.remaining_budget,
            cancellation_token=self._cancellation_token,
        )

        # Post-execution: update state
        wm.replace(driver_result.messages)
        safeguards.record_usage(
            steps=driver_result.steps_taken,
            cost=driver_result.cost_usd,
        )

        # Route lessons to memory system (if callback provided)
        if self._on_lessons and driver_result.lessons:
            self._on_lessons(driver_result.lessons, task_id)

        # Check hard cost ceiling
        if safeguards.hard_ceiling_exceeded:
            await self._emit_event("task.failed", task_id=task_id, trace_id=trace_id,
                                   reason="hard_cost_ceiling_exceeded")
            return TaskResult(
                status=DriverStatus.FAILED,
                task_id=task_id,
                summary="Hard cost ceiling exceeded",
            )

        # Handle result status
        status = driver_result.status

        if status == DriverStatus.COMPLETED:
            self._checkpoint_store.delete(task_id)
            await self._emit_event("task.completed", task_id=task_id, trace_id=trace_id)
            return TaskResult(
                status=status,
                task_id=task_id,
                summary=driver_result.summary,
                steps_taken=driver_result.steps_taken,
                lessons=driver_result.lessons,
                output_artifacts=driver_result.output_artifacts,
            )

        elif status == DriverStatus.FAILED:
            self._checkpoint_store.delete(task_id)
            await self._emit_event("task.failed", task_id=task_id, trace_id=trace_id)
            return TaskResult(
                status=status,
                task_id=task_id,
                summary=driver_result.summary,
                steps_taken=driver_result.steps_taken,
                lessons=driver_result.lessons,
            )

        elif status in (DriverStatus.ESCALATED, DriverStatus.BUDGET_EXHAUSTED):
            if not safeguards.can_escalate:
                await self._emit_event("task.failed", task_id=task_id, trace_id=trace_id,
                                       reason="max_escalations_exceeded")
                return TaskResult(
                    status=DriverStatus.FAILED,
                    task_id=task_id,
                    summary="Max escalations exceeded",
                )
            safeguards.record_escalation()
            now = datetime.now(timezone.utc).isoformat()
            self._save_checkpoint(task_id, wm, safeguards, "escalated", now, parent_task_id)

            # Publish escalation artifact
            if self._bus:
                art = create_artifact(
                    artifact_type=ArtifactType.ESCALATION,
                    sender=self._agent_id,
                    receiver="",  # ClusterRouter intercepts
                    task_id=task_id,
                    trace_id=trace_id,
                    summary=driver_result.escalation_reason or "Agent needs help",
                    payload={"reason": driver_result.escalation_reason},
                    parent_task_id=parent_task_id,
                )
                await self._bus.publish(art)

            await self._emit_event("task.suspended", task_id=task_id, trace_id=trace_id,
                                   reason=status.value)
            return TaskResult(
                status=status,
                task_id=task_id,
                escalation_reason=driver_result.escalation_reason,
                steps_taken=driver_result.steps_taken,
                lessons=driver_result.lessons,
            )

        elif status == DriverStatus.SUSPENDED:
            now = datetime.now(timezone.utc).isoformat()
            self._save_checkpoint(task_id, wm, safeguards, "suspended", now, parent_task_id)

            # Publish confirmation_request if present
            sc = driver_result.suspend_context
            if self._bus and sc and sc.confirmation_request:
                art = create_artifact(
                    artifact_type=ArtifactType.CONFIRMATION_REQUEST,
                    sender=self._agent_id,
                    receiver="",  # ClusterRouter intercepts
                    task_id=task_id,
                    trace_id=trace_id,
                    summary="High-risk tool confirmation needed",
                    payload=sc.confirmation_request,
                    parent_task_id=parent_task_id,
                )
                await self._bus.publish(art)

            await self._emit_event("task.suspended", task_id=task_id, trace_id=trace_id)
            return TaskResult(
                status=status,
                task_id=task_id,
                steps_taken=driver_result.steps_taken,
                lessons=driver_result.lessons,
            )

        # Should not reach here
        return TaskResult(status=DriverStatus.FAILED, task_id=task_id, summary="Unknown status")

    def _save_checkpoint(
        self,
        task_id: str,
        wm: WorkingMemory,
        safeguards: Safeguards,
        status: str,
        suspended_at: str,
        parent_task_id: str | None,
    ) -> None:
        cp = Checkpoint(
            schema_version=SCHEMA_VERSION,
            task_id=task_id,
            agent_id=self._agent_id,
            driver_type=type(self._driver).__name__,
            messages=wm.snapshot(),
            safeguards_state=safeguards.snapshot(),
            task_status=status,
            suspended_at=suspended_at,
            context_data={"parent_task_id": parent_task_id},
        )
        self._checkpoint_store.save(cp)

    async def _emit_event(self, event: str, **data: Any) -> None:
        await self._event_bus.emit(event, {"event": event, **data})
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/runner/test_task_runner.py -v`
Expected: 4 passed

- [ ] **Step 5: Commit**

```bash
git add src/hal/runner/task_runner.py tests/runner/test_task_runner.py
git commit -m "feat: add TaskRunner with run flow and lifecycle events"
```

---

## Task 6: TaskRunner — Suspend/Resume Flow

**Files:**
- Modify: `tests/runner/test_task_runner.py`

- [ ] **Step 1: Write the failing tests (append to test_task_runner.py)**

```python
# Append to tests/runner/test_task_runner.py

from hal.core.driver import SuspendContext


async def test_suspend_and_resume():
    driver = MockDriver(responses=[
        DriverResult(
            status=DriverStatus.SUSPENDED,
            messages=[{"role": "assistant", "content": "waiting"}],
            steps_taken=3,
            token_usage=400, cost_usd=0.4,
            suspend_context=SuspendContext(resume_event_id="evt-mr-123"),
        ),
        DriverResult(
            status=DriverStatus.COMPLETED,
            messages=[{"role": "assistant", "content": "done after resume"}],
            steps_taken=2,
            token_usage=200, cost_usd=0.2,
            summary="Completed after MR approval",
        ),
    ])
    role = _make_role()
    event_bus = AsyncEventBus()
    checkpoint_store = InMemoryCheckpointStore()

    runner = TaskRunner(
        agent_id="agent-1",
        driver=driver,
        role=role,
        event_bus=event_bus,
        checkpoint_store=checkpoint_store,
        hard_cost_ceiling=50.0,
    )

    # First run → SUSPENDED
    result = await runner.run(task_id="t-5", task_description="Wait for MR", trace_id="tr-5")
    assert result.status == DriverStatus.SUSPENDED

    # Checkpoint should exist
    cp = checkpoint_store.load("t-5")
    assert cp is not None
    assert cp.task_status == "suspended"

    # Resume with event data
    result = await runner.resume(
        task_id="t-5",
        trace_id="tr-5",
        event_data={"mr_status": "approved"},
    )
    assert result.status == DriverStatus.COMPLETED
    assert result.summary == "Completed after MR approval"

    # Checkpoint should be cleaned up
    assert checkpoint_store.load("t-5") is None


async def test_escalate_and_resume():
    driver = MockDriver(responses=[
        DriverResult(
            status=DriverStatus.ESCALATED,
            messages=[{"role": "assistant", "content": "stuck"}],
            steps_taken=5,
            token_usage=800, cost_usd=0.8,
            escalation_reason="Ambiguous API spec",
        ),
        DriverResult(
            status=DriverStatus.COMPLETED,
            messages=[{"role": "assistant", "content": "resolved"}],
            steps_taken=3,
            token_usage=500,
            cost_usd=0.5,
            summary="Issue resolved",
        ),
    ])
    role = _make_role()
    checkpoint_store = InMemoryCheckpointStore()

    runner = TaskRunner(
        agent_id="agent-1",
        driver=driver,
        role=role,
        event_bus=AsyncEventBus(),
        checkpoint_store=checkpoint_store,
        hard_cost_ceiling=50.0,
    )

    result = await runner.run(task_id="t-6", task_description="Clarify", trace_id="tr-6")
    assert result.status == DriverStatus.ESCALATED
    assert result.escalation_reason == "Ambiguous API spec"

    result = await runner.resume(
        task_id="t-6", trace_id="tr-6",
        event_data={"guidance": "Use v2 API"},
    )
    assert result.status == DriverStatus.COMPLETED


async def test_budget_exhausted():
    driver = MockDriver(responses=[
        DriverResult(
            status=DriverStatus.BUDGET_EXHAUSTED,
            messages=[],
            steps_taken=50,
            token_usage=5000, cost_usd=5.0,
        ),
    ])
    role = _make_role(max_steps=50, cost_budget=5.0)
    checkpoint_store = InMemoryCheckpointStore()

    runner = TaskRunner(
        agent_id="agent-1",
        driver=driver,
        role=role,
        event_bus=AsyncEventBus(),
        checkpoint_store=checkpoint_store,
        hard_cost_ceiling=50.0,
    )

    result = await runner.run(task_id="t-7", task_description="Expensive", trace_id="tr-7")
    assert result.status == DriverStatus.BUDGET_EXHAUSTED

    # Should be checkpointed for potential budget increase
    cp = checkpoint_store.load("t-7")
    assert cp is not None


async def test_max_escalations_exceeded():
    """After max_escalations, task FAILs instead of escalating."""
    responses = [
        DriverResult(status=DriverStatus.ESCALATED, messages=[], steps_taken=1,
                     token_usage=100, cost_usd=0.1, escalation_reason=f"help {i}")
        for i in range(4)  # max is 3
    ]
    driver = MockDriver(responses=responses)
    role = _make_role()
    role.safeguards.max_escalations_per_task = 3
    checkpoint_store = InMemoryCheckpointStore()

    runner = TaskRunner(
        agent_id="agent-1",
        driver=driver,
        role=role,
        event_bus=AsyncEventBus(),
        checkpoint_store=checkpoint_store,
        hard_cost_ceiling=50.0,
    )

    # First 3 escalations succeed
    for i in range(3):
        result = await runner.run(task_id=f"t-esc-{i}", task_description="Help", trace_id="tr")
        assert result.status == DriverStatus.ESCALATED

    # 4th escalation → FAILED (same runner, but safeguards track per-task)
    # Need a new runner to test max_escalations on a single task (sequential escalations)
    # Actually the runner creates fresh safeguards per run. For sequential escalation on same task,
    # we need to use resume flow.

    driver2 = MockDriver(responses=[
        DriverResult(status=DriverStatus.ESCALATED, messages=[{"role": "a", "content": "1"}],
                     steps_taken=1, token_usage=100, cost_usd=0.1, escalation_reason="help 1"),
        DriverResult(status=DriverStatus.ESCALATED, messages=[{"role": "a", "content": "2"}],
                     steps_taken=1, token_usage=100, cost_usd=0.1, escalation_reason="help 2"),
        DriverResult(status=DriverStatus.ESCALATED, messages=[{"role": "a", "content": "3"}],
                     steps_taken=1, token_usage=100, cost_usd=0.1, escalation_reason="help 3"),
        DriverResult(status=DriverStatus.ESCALATED, messages=[{"role": "a", "content": "4"}],
                     steps_taken=1, token_usage=100, cost_usd=0.1, escalation_reason="help 4"),
    ])
    checkpoint_store2 = InMemoryCheckpointStore()
    runner2 = TaskRunner(
        agent_id="agent-1",
        driver=driver2,
        role=role,
        event_bus=AsyncEventBus(),
        checkpoint_store=checkpoint_store2,
        hard_cost_ceiling=50.0,
    )

    r = await runner2.run(task_id="t-multi", task_description="Help", trace_id="tr")
    assert r.status == DriverStatus.ESCALATED  # 1st

    r = await runner2.resume(task_id="t-multi", trace_id="tr", event_data={"guidance": "try x"})
    assert r.status == DriverStatus.ESCALATED  # 2nd

    r = await runner2.resume(task_id="t-multi", trace_id="tr", event_data={"guidance": "try y"})
    assert r.status == DriverStatus.ESCALATED  # 3rd

    r = await runner2.resume(task_id="t-multi", trace_id="tr", event_data={"guidance": "try z"})
    assert r.status == DriverStatus.FAILED  # 4th → max exceeded
    assert "Max escalations" in r.summary


async def test_resume_nonexistent_task():
    driver = MockDriver(responses=[])
    role = _make_role()
    runner = TaskRunner(
        agent_id="agent-1",
        driver=driver,
        role=role,
        event_bus=AsyncEventBus(),
        checkpoint_store=InMemoryCheckpointStore(),
        hard_cost_ceiling=50.0,
    )

    result = await runner.resume(task_id="nonexistent", trace_id="tr", event_data={})
    assert result.status == DriverStatus.FAILED
    assert "No checkpoint" in result.summary
```

- [ ] **Step 2: Run test to verify new tests pass**

Run: `pytest tests/runner/test_task_runner.py -v`
Expected: 9 passed (4 previous + 5 new)

- [ ] **Step 3: Commit**

```bash
git add tests/runner/test_task_runner.py
git commit -m "test: add TaskRunner suspend/resume and escalation tests"
```

---

## Task 7: TaskRunner — Escalation Publishing

**Files:**
- Modify: `tests/runner/test_task_runner.py`

- [ ] **Step 1: Write the failing tests (append to test_task_runner.py)**

```python
# Append to tests/runner/test_task_runner.py

from hal.core.artifact import ArtifactType
from hal.core.bus.memory import InMemoryBus


async def test_escalation_publishes_artifact():
    driver = MockDriver(responses=[
        DriverResult(
            status=DriverStatus.ESCALATED,
            messages=[],
            steps_taken=2,
            token_usage=300,
            cost_usd=0.3,
            escalation_reason="Ambiguous spec",
        ),
    ])
    role = _make_role()
    event_bus = AsyncEventBus()
    checkpoint_store = InMemoryCheckpointStore()
    bus = InMemoryBus(max_inbox_size=10)

    published: list = []
    bus.subscribe("", lambda a: published.append(a))  # catch all

    runner = TaskRunner(
        agent_id="ios-dev-01",
        driver=driver,
        role=role,
        event_bus=event_bus,
        checkpoint_store=checkpoint_store,
        hard_cost_ceiling=50.0,
        bus=bus,
    )

    result = await runner.run(
        task_id="t-pub", task_description="Test publish", trace_id="tr-pub",
        parent_task_id="parent-1",
    )
    assert result.status == DriverStatus.ESCALATED
    assert len(published) == 1
    assert published[0].artifact_type == ArtifactType.ESCALATION
    assert published[0].parent_task_id == "parent-1"


async def test_confirmation_request_publishes_artifact():
    driver = MockDriver(responses=[
        DriverResult(
            status=DriverStatus.SUSPENDED,
            messages=[],
            steps_taken=1,
            token_usage=100,
            cost_usd=0.1,
            suspend_context=SuspendContext(
                confirmation_request={"tool_name": "deploy", "params": {"env": "prod"}},
            ),
        ),
    ])
    role = _make_role()
    bus = InMemoryBus(max_inbox_size=10)

    published: list = []
    bus.subscribe("", lambda a: published.append(a))

    runner = TaskRunner(
        agent_id="ios-dev-01",
        driver=driver,
        role=role,
        event_bus=AsyncEventBus(),
        checkpoint_store=InMemoryCheckpointStore(),
        hard_cost_ceiling=50.0,
        bus=bus,
    )

    result = await runner.run(
        task_id="t-conf", task_description="Deploy", trace_id="tr-conf",
    )
    assert result.status == DriverStatus.SUSPENDED
    assert len(published) == 1
    assert published[0].artifact_type == ArtifactType.CONFIRMATION_REQUEST
```

- [ ] **Step 2: Run tests**

Run: `pytest tests/runner/test_task_runner.py -v`
Expected: 11 passed

- [ ] **Step 3: Commit**

```bash
git add tests/runner/test_task_runner.py
git commit -m "test: add escalation and confirmation artifact publishing tests"
```

---

## Task 8: Suspend TTL

**Files:**
- Modify: `tests/runner/test_task_runner.py`

- [ ] **Step 1: Write the failing tests (append to test_task_runner.py)**

```python
# Append to tests/runner/test_task_runner.py

from datetime import datetime, timedelta, timezone
from hal.runner.task_runner import check_suspend_ttl
from hal.runner.checkpoint import Checkpoint, SCHEMA_VERSION


def test_check_suspend_ttl_not_expired():
    now = datetime.now(timezone.utc)
    cp = Checkpoint(
        schema_version=SCHEMA_VERSION,
        task_id="t-ttl-1",
        agent_id="a",
        driver_type="MockDriver",
        messages=[],
        safeguards_state={"steps_used": 0, "cost_used": 0.0, "escalations_used": 0},
        task_status="suspended",
        suspended_at=(now - timedelta(hours=1)).isoformat(),
    )
    assert check_suspend_ttl(cp, ttl_seconds=86400) is False  # not expired


def test_check_suspend_ttl_expired():
    now = datetime.now(timezone.utc)
    cp = Checkpoint(
        schema_version=SCHEMA_VERSION,
        task_id="t-ttl-2",
        agent_id="a",
        driver_type="MockDriver",
        messages=[],
        safeguards_state={"steps_used": 0, "cost_used": 0.0, "escalations_used": 0},
        task_status="suspended",
        suspended_at=(now - timedelta(hours=25)).isoformat(),
    )
    assert check_suspend_ttl(cp, ttl_seconds=86400) is True  # expired


def test_check_suspend_ttl_no_suspended_at():
    cp = Checkpoint(
        schema_version=SCHEMA_VERSION,
        task_id="t-ttl-3",
        agent_id="a",
        driver_type="MockDriver",
        messages=[],
        safeguards_state={"steps_used": 0, "cost_used": 0.0, "escalations_used": 0},
        task_status="suspended",
        suspended_at=None,
    )
    assert check_suspend_ttl(cp, ttl_seconds=86400) is False
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/runner/test_task_runner.py::test_check_suspend_ttl_not_expired -v`
Expected: FAIL with `ImportError`

- [ ] **Step 3: Add check_suspend_ttl to task_runner.py**

Add this function to `src/hal/runner/task_runner.py`:

```python
def check_suspend_ttl(checkpoint: Checkpoint, ttl_seconds: int) -> bool:
    """Check if a suspended checkpoint has exceeded its TTL.

    Returns True if expired, False otherwise.
    Used by AgentService's periodic TTL scan.
    """
    if checkpoint.suspended_at is None:
        return False

    suspended_at = datetime.fromisoformat(checkpoint.suspended_at)
    elapsed = (datetime.now(timezone.utc) - suspended_at).total_seconds()
    return elapsed > ttl_seconds
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/runner/test_task_runner.py -v`
Expected: 14 passed

- [ ] **Step 5: Commit**

```bash
git add src/hal/runner/task_runner.py tests/runner/test_task_runner.py
git commit -m "feat: add suspend TTL check for checkpoint expiration"
```

---

## Task 9: Integration Test

**Files:**
- Create: `tests/runner/test_runner_integration.py`

- [ ] **Step 1: Write the integration test**

`tests/runner/test_runner_integration.py`:
```python
"""Integration: full lifecycle — run → suspend → resume → complete."""

import pytest

from hal.core.bus.memory import InMemoryBus
from hal.core.driver import DriverResult, DriverStatus, SuspendContext
from hal.core.event_bus import AsyncEventBus
from hal.drivers.mock import MockDriver
from hal.identity.role import IdentityConfig, RoleProfile, SafeguardConfig
from hal.runner.checkpoint import InMemoryCheckpointStore
from hal.runner.task_runner import TaskRunner


async def test_full_lifecycle():
    """run → suspend (PENDING tool) → resume → complete."""
    driver = MockDriver(responses=[
        DriverResult(
            status=DriverStatus.SUSPENDED,
            messages=[
                {"role": "user", "content": "Submit MR"},
                {"role": "assistant", "content": "MR submitted, waiting for review"},
            ],
            steps_taken=5,
            token_usage=800, cost_usd=0.8,
            suspend_context=SuspendContext(resume_event_id="evt-mr-456"),
        ),
        DriverResult(
            status=DriverStatus.COMPLETED,
            messages=[
                {"role": "user", "content": "Submit MR"},
                {"role": "assistant", "content": "MR submitted, waiting for review"},
                {"role": "user", "content": "[RESUME EVENT] {'mr_status': 'approved'}"},
                {"role": "assistant", "content": "MR approved, merging..."},
            ],
            steps_taken=3,
            token_usage=400, cost_usd=0.4,
            summary="MR merged successfully",
        ),
    ])

    role = RoleProfile(
        identity=IdentityConfig(name="iOS Dev", persona="You are an iOS developer."),
        safeguards=SafeguardConfig(max_steps=100, cost_budget=10.0),
    )
    event_bus = AsyncEventBus()
    checkpoint_store = InMemoryCheckpointStore()
    bus = InMemoryBus(max_inbox_size=10)

    events: list[dict] = []
    async def event_handler(data: dict) -> None:
        events.append(data)
    event_bus.subscribe("task.*", event_handler)

    runner = TaskRunner(
        agent_id="ios-dev-01",
        driver=driver,
        role=role,
        event_bus=event_bus,
        checkpoint_store=checkpoint_store,
        hard_cost_ceiling=50.0,
        bus=bus,
    )

    # Run → SUSPENDED
    result = await runner.run(
        task_id="task-full",
        task_description="Submit and merge MR",
        trace_id="trace-full",
        parent_task_id="parent-main",
    )
    assert result.status == DriverStatus.SUSPENDED
    assert result.steps_taken == 5

    # Verify checkpoint saved
    cp = checkpoint_store.load("task-full")
    assert cp is not None
    assert cp.task_status == "suspended"
    assert cp.context_data["parent_task_id"] == "parent-main"

    # Verify safeguards state in checkpoint
    assert cp.safeguards_state["steps_used"] == 5
    assert cp.safeguards_state["cost_used"] == 0.8  # cost_usd=0.8

    # Resume → COMPLETED
    result = await runner.resume(
        task_id="task-full",
        trace_id="trace-full",
        event_data={"mr_status": "approved"},
    )
    assert result.status == DriverStatus.COMPLETED
    assert result.summary == "MR merged successfully"

    # Checkpoint cleaned up
    assert checkpoint_store.load("task-full") is None

    # Verify lifecycle events
    event_types = [e["event"] for e in events]
    assert "task.started" in event_types
    assert "task.suspended" in event_types
    assert "task.resumed" in event_types
    assert "task.completed" in event_types
```

- [ ] **Step 2: Run integration test**

Run: `pytest tests/runner/test_runner_integration.py -v`
Expected: 1 passed

- [ ] **Step 3: Run full test suite**

Run: `pytest tests/ -v`
Expected: All tests pass

- [ ] **Step 4: Commit**

```bash
git add tests/runner/test_runner_integration.py
git commit -m "test: add TaskRunner full lifecycle integration test"
```

---

## Summary

| Task | Component | Test count (est.) |
|------|-----------|-------------------|
| 1 | Working Memory | 6 |
| 2 | MockDriver | 7 |
| 3 | Safeguards | 6 |
| 4 | Checkpoint (InMemory + SQLite) | 11 |
| 5 | TaskRunner — run flow | 4 |
| 6 | TaskRunner — suspend/resume | 5 |
| 7 | TaskRunner — artifact publishing | 2 |
| 8 | Suspend TTL | 3 |
| 9 | Integration | 1 |
| **Total** | | **~45** |
