# Plan 1: Core Kernel — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build HAL's foundation layer — config loading, async event bus, structured logging, plugin registry, error classification, AgentDriver protocol, Artifact model, and HAL Bus (InMemoryBus backend).

**Architecture:** Bottom-up construction of Layer 1. All components are decoupled through protocols and the event bus. The Bus uses a pluggable backend pattern (Protocol → InMemoryBus). Artifacts are immutable dataclasses with authority enforcement. The AgentDriver is a pure protocol — no implementation in this plan (MockDriver is in Plan 4).

**Tech Stack:** Python 3.12+, asyncio, dataclasses, PyYAML, pytest, pytest-asyncio

**Spec:** `docs/superpowers/specs/2026-04-27-hal-design.md` — Layer 1: Core Kernel

---

## File Structure

```
hal/
├── pyproject.toml
├── src/
│   └── hal/
│       ├── __init__.py
│       └── core/
│           ├── __init__.py
│           ├── errors.py              # Error classification (Recoverable/Unrecoverable/EscalateToHuman)
│           ├── config.py              # Config loader (YAML/JSON, env var override, layered merge)
│           ├── event_bus.py           # AsyncEventBus (pub/sub, wildcard, fault isolation)
│           ├── logger.py              # Structured JSON logger with trace context
│           ├── plugin_registry.py     # Central plugin registry (category → name → plugin)
│           ├── artifact.py            # Artifact model (immutable, authority-stamped)
│           ├── driver.py              # AgentDriver protocol + DriverResult
│           └── bus/
│               ├── __init__.py
│               ├── protocol.py        # Bus protocol (abstract interface)
│               └── memory.py          # InMemoryBus implementation
└── tests/
    ├── conftest.py
    └── core/
        ├── __init__.py
        ├── test_errors.py
        ├── test_config.py
        ├── test_event_bus.py
        ├── test_logger.py
        ├── test_plugin_registry.py
        ├── test_artifact.py
        ├── test_driver.py
        ├── bus/
        │   ├── __init__.py
        │   └── test_memory_bus.py
        └── test_integration.py
```

---

## Task 1: Project Scaffolding

**Files:**
- Create: `pyproject.toml`
- Create: `src/hal/__init__.py`
- Create: `src/hal/core/__init__.py`
- Create: `src/hal/core/bus/__init__.py`
- Create: `tests/conftest.py`
- Create: `tests/core/__init__.py`
- Create: `tests/core/bus/__init__.py`

- [ ] **Step 1: Create pyproject.toml**

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "hal"
version = "0.1.0"
description = "HAL — A general-purpose AI agent coordination framework"
requires-python = ">=3.12"
dependencies = [
    "pyyaml>=6.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=0.24",
    "ruff>=0.4",
]

[tool.hatch.build.targets.wheel]
packages = ["src/hal"]

[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"

[tool.ruff]
target-version = "py312"
line-length = 100
```

- [ ] **Step 2: Create package __init__ files**

`src/hal/__init__.py`:
```python
"""HAL — A general-purpose AI agent coordination framework."""
```

`src/hal/core/__init__.py`:
```python
"""HAL Core Kernel — foundation infrastructure for all upper layers."""
```

`src/hal/core/bus/__init__.py`:
```python
"""HAL Bus — async artifact transport between agent instances."""
```

`tests/conftest.py`:
```python
"""Shared test fixtures for HAL."""
```

`tests/core/__init__.py` and `tests/core/bus/__init__.py`: empty files.

- [ ] **Step 3: Install project in dev mode and verify**

Run: `cd /Users/altair/Documents/Projects/HAL && python -m pip install -e ".[dev]"`
Expected: Successful installation.

Run: `python -c "import hal; print('OK')"`
Expected: `OK`

- [ ] **Step 4: Verify pytest runs**

Run: `pytest --co`
Expected: `no tests ran` (no test files yet, no errors)

- [ ] **Step 5: Commit**

```bash
git init
git add pyproject.toml src/ tests/
git commit -m "chore: scaffold HAL project structure"
```

---

## Task 2: Error Classification

**Files:**
- Create: `src/hal/core/errors.py`
- Create: `tests/core/test_errors.py`

- [ ] **Step 1: Write the failing test**

`tests/core/test_errors.py`:
```python
from hal.core.errors import HALError, Recoverable, Unrecoverable, EscalateToHuman


def test_recoverable_is_hal_error():
    err = Recoverable("temporary failure")
    assert isinstance(err, HALError)
    assert str(err) == "temporary failure"


def test_unrecoverable_is_hal_error():
    err = Unrecoverable("fatal crash")
    assert isinstance(err, HALError)
    assert str(err) == "fatal crash"


def test_escalate_to_human_carries_context():
    err = EscalateToHuman("ambiguous requirement", context={"task_id": "t-1"})
    assert isinstance(err, HALError)
    assert err.context == {"task_id": "t-1"}
    assert str(err) == "ambiguous requirement"


def test_error_hierarchy_is_distinct():
    """Each error type is catchable independently."""
    recoverable = Recoverable("r")
    unrecoverable = Unrecoverable("u")
    escalate = EscalateToHuman("e")

    assert not isinstance(recoverable, Unrecoverable)
    assert not isinstance(unrecoverable, Recoverable)
    assert not isinstance(escalate, Recoverable)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/core/test_errors.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`src/hal/core/errors.py`:
```python
"""HAL error classification.

Three error categories:
- Recoverable: transient failures, safe to retry.
- Unrecoverable: fatal errors, task must abort.
- EscalateToHuman: needs human decision to proceed.
"""

from typing import Any


class HALError(Exception):
    """Base exception for all HAL errors."""


class Recoverable(HALError):
    """Transient failure — retry is safe."""


class Unrecoverable(HALError):
    """Fatal error — task must abort."""


class EscalateToHuman(HALError):
    """Requires human decision to proceed."""

    def __init__(self, message: str, context: dict[str, Any] | None = None) -> None:
        super().__init__(message)
        self.context = context or {}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/core/test_errors.py -v`
Expected: 4 passed

- [ ] **Step 5: Commit**

```bash
git add src/hal/core/errors.py tests/core/test_errors.py
git commit -m "feat: add error classification (Recoverable/Unrecoverable/EscalateToHuman)"
```

---

## Task 3: Config Loader

**Files:**
- Create: `src/hal/core/config.py`
- Create: `tests/core/test_config.py`

- [ ] **Step 1: Write the failing tests**

`tests/core/test_config.py`:
```python
import os
import json
import tempfile
from pathlib import Path

import pytest
import yaml

from hal.core.config import load_config, merge_configs


def test_load_yaml_config(tmp_path: Path):
    config_file = tmp_path / "config.yaml"
    config_file.write_text(yaml.dump({"cluster": {"name": "test-team"}}))

    config = load_config(config_file)
    assert config["cluster"]["name"] == "test-team"


def test_load_json_config(tmp_path: Path):
    config_file = tmp_path / "config.json"
    config_file.write_text(json.dumps({"cluster": {"name": "json-team"}}))

    config = load_config(config_file)
    assert config["cluster"]["name"] == "json-team"


def test_env_var_override(tmp_path: Path, monkeypatch: pytest.MonkeyPatch):
    monkeypatch.setenv("HAL_CLUSTER_NAME", "env-team")
    config_file = tmp_path / "config.yaml"
    config_file.write_text(yaml.dump({"cluster": {"name": "${HAL_CLUSTER_NAME}"}}))

    config = load_config(config_file)
    assert config["cluster"]["name"] == "env-team"


def test_env_var_missing_raises(tmp_path: Path):
    config_file = tmp_path / "config.yaml"
    config_file.write_text(yaml.dump({"key": "${NONEXISTENT_VAR_12345}"}))

    with pytest.raises(ValueError, match="NONEXISTENT_VAR_12345"):
        load_config(config_file)


def test_merge_configs_deep():
    base = {"a": {"b": 1, "c": 2}, "d": 3}
    override = {"a": {"b": 10, "e": 5}, "f": 6}

    merged = merge_configs(base, override)
    assert merged == {"a": {"b": 10, "c": 2, "e": 5}, "d": 3, "f": 6}


def test_merge_configs_override_replaces_non_dict():
    base = {"a": {"b": 1}}
    override = {"a": "replaced"}

    merged = merge_configs(base, override)
    assert merged == {"a": "replaced"}


def test_load_config_file_not_found():
    with pytest.raises(FileNotFoundError):
        load_config(Path("/nonexistent/path/config.yaml"))


def test_load_config_unsupported_extension(tmp_path: Path):
    config_file = tmp_path / "config.toml"
    config_file.write_text("key = 'value'")

    with pytest.raises(ValueError, match="Unsupported"):
        load_config(config_file)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/core/test_config.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`src/hal/core/config.py`:
```python
"""Config loader — YAML/JSON with env var overrides and layered merge."""

import json
import os
import re
from pathlib import Path
from typing import Any

import yaml

_ENV_VAR_PATTERN = re.compile(r"\$\{([^}]+)\}")


def load_config(path: Path) -> dict[str, Any]:
    """Load a config file (YAML or JSON), resolving ${ENV_VAR} references."""
    if not path.exists():
        raise FileNotFoundError(f"Config file not found: {path}")

    text = path.read_text(encoding="utf-8")
    suffix = path.suffix.lower()

    if suffix in (".yaml", ".yml"):
        raw = yaml.safe_load(text) or {}
    elif suffix == ".json":
        raw = json.loads(text)
    else:
        raise ValueError(f"Unsupported config file extension: {suffix}")

    return _resolve_env_vars(raw)


def merge_configs(base: dict[str, Any], override: dict[str, Any]) -> dict[str, Any]:
    """Deep-merge override into base. Override wins on conflict."""
    merged = base.copy()
    for key, value in override.items():
        if key in merged and isinstance(merged[key], dict) and isinstance(value, dict):
            merged[key] = merge_configs(merged[key], value)
        else:
            merged[key] = value
    return merged


def _resolve_env_vars(obj: Any) -> Any:
    """Recursively resolve ${ENV_VAR} references in config values."""
    if isinstance(obj, str):
        match = _ENV_VAR_PATTERN.fullmatch(obj)
        if match:
            var_name = match.group(1)
            value = os.environ.get(var_name)
            if value is None:
                raise ValueError(
                    f"Environment variable '{var_name}' referenced in config but not set"
                )
            return value
        # Also handle env vars embedded in longer strings
        def _replace(m: re.Match) -> str:
            var_name = m.group(1)
            value = os.environ.get(var_name)
            if value is None:
                raise ValueError(
                    f"Environment variable '{var_name}' referenced in config but not set"
                )
            return value

        return _ENV_VAR_PATTERN.sub(_replace, obj)
    elif isinstance(obj, dict):
        return {k: _resolve_env_vars(v) for k, v in obj.items()}
    elif isinstance(obj, list):
        return [_resolve_env_vars(item) for item in obj]
    return obj
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/core/test_config.py -v`
Expected: 8 passed

- [ ] **Step 5: Commit**

```bash
git add src/hal/core/config.py tests/core/test_config.py
git commit -m "feat: add config loader with YAML/JSON support and env var overrides"
```

---

## Task 4: AsyncEventBus

**Files:**
- Create: `src/hal/core/event_bus.py`
- Create: `tests/core/test_event_bus.py`

- [ ] **Step 1: Write the failing tests**

`tests/core/test_event_bus.py`:
```python
import asyncio
import logging

import pytest

from hal.core.event_bus import AsyncEventBus


@pytest.fixture
def bus() -> AsyncEventBus:
    return AsyncEventBus()


async def test_subscribe_and_emit(bus: AsyncEventBus):
    received: list[dict] = []

    async def handler(data: dict) -> None:
        received.append(data)

    bus.subscribe("tool.called", handler)
    await bus.emit("tool.called", {"tool": "git"})

    assert received == [{"tool": "git"}]


async def test_multiple_handlers(bus: AsyncEventBus):
    results: list[str] = []

    async def handler_a(data: dict) -> None:
        results.append("a")

    async def handler_b(data: dict) -> None:
        results.append("b")

    bus.subscribe("task.started", handler_a)
    bus.subscribe("task.started", handler_b)
    await bus.emit("task.started", {})

    assert sorted(results) == ["a", "b"]


async def test_wildcard_subscription(bus: AsyncEventBus):
    received: list[str] = []

    async def handler(data: dict) -> None:
        received.append(data["event"])

    bus.subscribe("tool.*", handler)
    await bus.emit("tool.called", {"event": "tool.called"})
    await bus.emit("tool.failed", {"event": "tool.failed"})
    await bus.emit("task.started", {"event": "task.started"})  # should NOT match

    assert received == ["tool.called", "tool.failed"]


async def test_no_matching_subscribers(bus: AsyncEventBus):
    # Should not raise — just a no-op
    await bus.emit("unknown.event", {"key": "value"})


async def test_handler_failure_isolated(bus: AsyncEventBus, caplog):
    results: list[str] = []

    async def bad_handler(data: dict) -> None:
        raise RuntimeError("handler crashed")

    async def good_handler(data: dict) -> None:
        results.append("ok")

    bus.subscribe("test.event", bad_handler)
    bus.subscribe("test.event", good_handler)

    with caplog.at_level(logging.ERROR):
        await bus.emit("test.event", {})

    assert results == ["ok"]
    assert "handler crashed" in caplog.text


async def test_unsubscribe(bus: AsyncEventBus):
    received: list[dict] = []

    async def handler(data: dict) -> None:
        received.append(data)

    bus.subscribe("evt", handler)
    await bus.emit("evt", {"n": 1})
    bus.unsubscribe("evt", handler)
    await bus.emit("evt", {"n": 2})

    assert received == [{"n": 1}]
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/core/test_event_bus.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`src/hal/core/event_bus.py`:
```python
"""AsyncEventBus — internal async pub/sub with wildcard support and fault isolation."""

import fnmatch
import logging
from collections import defaultdict
from collections.abc import Awaitable, Callable
from typing import Any

logger = logging.getLogger(__name__)

EventHandler = Callable[[dict[str, Any]], Awaitable[None]]


class AsyncEventBus:
    """Async event bus with wildcard subscriptions.

    Patterns use fnmatch-style matching (e.g., "tool.*" matches "tool.called").
    A failing handler is logged but does not prevent other handlers from running.
    """

    def __init__(self) -> None:
        self._handlers: dict[str, list[EventHandler]] = defaultdict(list)

    def subscribe(self, pattern: str, handler: EventHandler) -> None:
        """Register a handler for events matching the pattern."""
        self._handlers[pattern].append(handler)

    def unsubscribe(self, pattern: str, handler: EventHandler) -> None:
        """Remove a handler from a pattern."""
        handlers = self._handlers.get(pattern)
        if handlers and handler in handlers:
            handlers.remove(handler)

    async def emit(self, event: str, data: dict[str, Any]) -> None:
        """Emit an event to all matching handlers. Failures are logged, not raised."""
        for pattern, handlers in self._handlers.items():
            if fnmatch.fnmatch(event, pattern):
                for handler in handlers:
                    try:
                        await handler(data)
                    except Exception:
                        logger.exception(
                            "Event handler failed for event '%s' (pattern '%s')",
                            event,
                            pattern,
                        )
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/core/test_event_bus.py -v`
Expected: 6 passed

- [ ] **Step 5: Commit**

```bash
git add src/hal/core/event_bus.py tests/core/test_event_bus.py
git commit -m "feat: add AsyncEventBus with wildcard subscriptions and fault isolation"
```

---

## Task 5: Structured Logger

**Files:**
- Create: `src/hal/core/logger.py`
- Create: `tests/core/test_logger.py`

- [ ] **Step 1: Write the failing tests**

`tests/core/test_logger.py`:
```python
import json
import logging

import pytest

from hal.core.logger import HALLogger, configure_logging


@pytest.fixture
def hal_logger() -> HALLogger:
    return HALLogger(agent_id="ios-dev-01")


def test_log_entry_contains_agent_id(hal_logger: HALLogger, capfd):
    configure_logging()
    hal_logger.info("task started", task_id="t-1", trace_id="tr-abc")

    captured = capfd.readouterr()
    entry = json.loads(captured.err.strip().split("\n")[-1])
    assert entry["agent_id"] == "ios-dev-01"
    assert entry["task_id"] == "t-1"
    assert entry["trace_id"] == "tr-abc"
    assert entry["message"] == "task started"
    assert "timestamp" in entry


def test_log_entry_with_extra_data(hal_logger: HALLogger, capfd):
    configure_logging()
    hal_logger.info("tool called", task_id="t-2", trace_id="tr-def", tool="git", action="commit")

    captured = capfd.readouterr()
    entry = json.loads(captured.err.strip().split("\n")[-1])
    assert entry["tool"] == "git"
    assert entry["action"] == "commit"


def test_log_levels(hal_logger: HALLogger, capfd):
    configure_logging(level=logging.WARNING)
    hal_logger.info("should not appear", task_id="t-3", trace_id="tr-ghi")
    hal_logger.warning("should appear", task_id="t-3", trace_id="tr-ghi")

    captured = capfd.readouterr()
    lines = [line for line in captured.err.strip().split("\n") if line]
    assert len(lines) == 1
    entry = json.loads(lines[0])
    assert entry["level"] == "WARNING"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/core/test_logger.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`src/hal/core/logger.py`:
```python
"""Structured JSON logger with trace context for HAL agents."""

import json
import logging
import sys
from datetime import datetime, timezone
from typing import Any


class _JSONFormatter(logging.Formatter):
    """Format log records as single-line JSON."""

    def format(self, record: logging.LogRecord) -> str:
        entry: dict[str, Any] = {
            "timestamp": datetime.now(timezone.utc).isoformat(),
            "level": record.levelname,
            "message": record.getMessage(),
        }
        # Merge HAL-specific context attached by HALLogger
        if hasattr(record, "hal_context"):
            entry.update(record.hal_context)
        return json.dumps(entry, ensure_ascii=False)


def configure_logging(level: int = logging.INFO) -> None:
    """Configure the root HAL logger to output JSON to stderr."""
    root = logging.getLogger("hal")
    # Avoid adding duplicate handlers
    if not root.handlers:
        handler = logging.StreamHandler(sys.stderr)
        handler.setFormatter(_JSONFormatter())
        root.addHandler(handler)
    root.setLevel(level)


class HALLogger:
    """Structured logger that injects agent_id and trace context into every entry."""

    def __init__(self, agent_id: str) -> None:
        self._agent_id = agent_id
        self._logger = logging.getLogger(f"hal.agent.{agent_id}")

    def _log(
        self,
        level: int,
        message: str,
        task_id: str,
        trace_id: str,
        **extra: Any,
    ) -> None:
        context = {
            "agent_id": self._agent_id,
            "task_id": task_id,
            "trace_id": trace_id,
            **extra,
        }
        record = self._logger.makeRecord(
            name=self._logger.name,
            level=level,
            fn="",
            lno=0,
            msg=message,
            args=(),
            exc_info=None,
        )
        record.hal_context = context  # type: ignore[attr-defined]
        self._logger.handle(record)

    def info(self, message: str, *, task_id: str, trace_id: str, **extra: Any) -> None:
        self._log(logging.INFO, message, task_id, trace_id, **extra)

    def warning(self, message: str, *, task_id: str, trace_id: str, **extra: Any) -> None:
        self._log(logging.WARNING, message, task_id, trace_id, **extra)

    def error(self, message: str, *, task_id: str, trace_id: str, **extra: Any) -> None:
        self._log(logging.ERROR, message, task_id, trace_id, **extra)

    def debug(self, message: str, *, task_id: str, trace_id: str, **extra: Any) -> None:
        self._log(logging.DEBUG, message, task_id, trace_id, **extra)
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/core/test_logger.py -v`
Expected: 3 passed

- [ ] **Step 5: Commit**

```bash
git add src/hal/core/logger.py tests/core/test_logger.py
git commit -m "feat: add structured JSON logger with trace context"
```

---

## Task 6: Plugin Registry

**Files:**
- Create: `src/hal/core/plugin_registry.py`
- Create: `tests/core/test_plugin_registry.py`

- [ ] **Step 1: Write the failing tests**

`tests/core/test_plugin_registry.py`:
```python
import pytest

from hal.core.plugin_registry import PluginRegistry


@pytest.fixture
def registry() -> PluginRegistry:
    return PluginRegistry()


def test_register_and_get(registry: PluginRegistry):
    plugin = {"type": "inmemory"}
    registry.register("storage", "inmemory", plugin)
    assert registry.get("storage", "inmemory") is plugin


def test_get_nonexistent_returns_none(registry: PluginRegistry):
    assert registry.get("storage", "redis") is None


def test_list_category(registry: PluginRegistry):
    registry.register("transport", "inmemory", object())
    registry.register("transport", "redis", object())
    registry.register("storage", "sqlite", object())

    names = registry.list_category("transport")
    assert sorted(names) == ["inmemory", "redis"]


def test_list_empty_category(registry: PluginRegistry):
    assert registry.list_category("nonexistent") == []


def test_duplicate_register_raises(registry: PluginRegistry):
    registry.register("storage", "inmemory", object())
    with pytest.raises(ValueError, match="already registered"):
        registry.register("storage", "inmemory", object())


def test_register_multiple_categories(registry: PluginRegistry):
    a = object()
    b = object()
    registry.register("storage", "sqlite", a)
    registry.register("transport", "sqlite", b)

    assert registry.get("storage", "sqlite") is a
    assert registry.get("transport", "sqlite") is b
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/core/test_plugin_registry.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`src/hal/core/plugin_registry.py`:
```python
"""Central plugin registry — category-scoped named plugin storage."""

from collections import defaultdict
from typing import Any


class PluginRegistry:
    """Registry for named plugins under categories.

    Categories include: tools, roles, storage, transport.
    Each (category, name) pair maps to exactly one plugin instance.
    """

    def __init__(self) -> None:
        self._plugins: dict[str, dict[str, Any]] = defaultdict(dict)

    def register(self, category: str, name: str, plugin: Any) -> None:
        """Register a plugin. Raises ValueError if (category, name) already exists."""
        if name in self._plugins[category]:
            raise ValueError(
                f"Plugin '{name}' already registered under category '{category}'"
            )
        self._plugins[category][name] = plugin

    def get(self, category: str, name: str) -> Any | None:
        """Get a plugin by category and name. Returns None if not found."""
        return self._plugins.get(category, {}).get(name)

    def list_category(self, category: str) -> list[str]:
        """List all plugin names in a category."""
        return list(self._plugins.get(category, {}).keys())
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/core/test_plugin_registry.py -v`
Expected: 6 passed

- [ ] **Step 5: Commit**

```bash
git add src/hal/core/plugin_registry.py tests/core/test_plugin_registry.py
git commit -m "feat: add PluginRegistry for category-scoped plugin management"
```

---

## Task 7: Artifact Model

**Files:**
- Create: `src/hal/core/artifact.py`
- Create: `tests/core/test_artifact.py`

- [ ] **Step 1: Write the failing tests**

`tests/core/test_artifact.py`:
```python
import pytest

from hal.core.artifact import Artifact, ArtifactType, Authority, create_artifact


def test_create_artifact_defaults():
    art = create_artifact(
        artifact_type=ArtifactType.TASK_ASSIGNMENT,
        sender="pm-01",
        receiver="ios-dev-01",
        task_id="task-1",
        trace_id="trace-abc",
        summary="Implement login SDK",
        payload={"description": "Build OAuth login"},
    )

    assert art.artifact_type == ArtifactType.TASK_ASSIGNMENT
    assert art.sender == "pm-01"
    assert art.receiver == "ios-dev-01"
    assert art.task_id == "task-1"
    assert art.trace_id == "trace-abc"
    assert art.authority == Authority.MEMBER  # default
    assert art.parent_task_id is None
    assert art.reply_to is None
    assert art.resume_event is None
    assert art.artifact_id  # auto-generated, non-empty


def test_artifact_is_immutable():
    art = create_artifact(
        artifact_type=ArtifactType.ESCALATION,
        sender="ios-dev-01",
        receiver="pm-01",
        task_id="task-2",
        trace_id="trace-def",
        summary="Need help",
        payload={},
    )
    with pytest.raises(AttributeError):
        art.summary = "changed"  # type: ignore[misc]


def test_artifact_with_parent_task_id():
    art = create_artifact(
        artifact_type=ArtifactType.TASK_ASSIGNMENT,
        sender="pm-01",
        receiver="ios-dev-01",
        task_id="sub-task-1",
        trace_id="trace-ghi",
        summary="Sub-task",
        payload={},
        parent_task_id="parent-task-1",
    )
    assert art.parent_task_id == "parent-task-1"


def test_artifact_authority_enforced_as_member():
    """create_artifact always sets authority=MEMBER. Only ClusterRouter sets HUMAN."""
    art = create_artifact(
        artifact_type=ArtifactType.RESUME_EVENT,
        sender="pm-01",
        receiver="ios-dev-01",
        task_id="task-3",
        trace_id="trace-jkl",
        summary="Resume",
        payload={},
    )
    assert art.authority == Authority.MEMBER


def test_artifact_types_comprehensive():
    """Verify all artifact types from spec are defined."""
    expected = {
        "TASK_ASSIGNMENT",
        "RESUME_EVENT",
        "ESCALATION",
        "CONFIRMATION_REQUEST",
        "TASK_STATUS_UPDATE",
        "DECISION",
        "CODE_CHANGE",
        "DESIGN_DOC",
        "TEST_REPORT",
        "REVIEW_FEEDBACK",
    }
    actual = {t.name for t in ArtifactType}
    assert expected.issubset(actual)


def test_artifact_serialization_roundtrip():
    art = create_artifact(
        artifact_type=ArtifactType.CODE_CHANGE,
        sender="ios-dev-01",
        receiver="pm-01",
        task_id="task-4",
        trace_id="trace-mno",
        summary="Branch pushed",
        payload={"branch": "feat/login", "files_changed": 3},
        parent_task_id="parent-1",
    )

    data = art.to_dict()
    restored = Artifact.from_dict(data)

    assert restored.artifact_id == art.artifact_id
    assert restored.artifact_type == art.artifact_type
    assert restored.sender == art.sender
    assert restored.payload == art.payload
    assert restored.parent_task_id == art.parent_task_id
    assert restored.authority == art.authority
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/core/test_artifact.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`src/hal/core/artifact.py`:
```python
"""Artifact model — immutable, authority-stamped deliverable between agents."""

import uuid
from dataclasses import dataclass, field, fields
from datetime import datetime, timezone
from enum import Enum
from typing import Any


class ArtifactType(Enum):
    """All artifact types defined in the HAL spec."""

    TASK_ASSIGNMENT = "task_assignment"
    RESUME_EVENT = "resume_event"
    ESCALATION = "escalation"
    CONFIRMATION_REQUEST = "confirmation_request"
    TASK_STATUS_UPDATE = "task_status_update"
    DECISION = "decision"
    CODE_CHANGE = "code_change"
    DESIGN_DOC = "design_doc"
    TEST_REPORT = "test_report"
    REVIEW_FEEDBACK = "review_feedback"


class Authority(Enum):
    """Trust level of the artifact source.

    MEMBER: from a peer or subordinate agent (default, enforced by Bus).
    HUMAN: from Human (only ClusterRouter can set this).
    """

    MEMBER = "member"
    HUMAN = "human"


@dataclass(frozen=True)
class Artifact:
    """Immutable deliverable passed between agents via HAL Bus."""

    artifact_id: str
    artifact_type: ArtifactType
    sender: str
    receiver: str
    task_id: str
    trace_id: str
    summary: str
    payload: dict[str, Any]
    authority: Authority
    created_at: str
    parent_task_id: str | None = None
    reply_to: str | None = None
    resume_event: str | None = None

    def to_dict(self) -> dict[str, Any]:
        """Serialize to a plain dict for storage/transport."""
        data: dict[str, Any] = {}
        for f in fields(self):
            value = getattr(self, f.name)
            if isinstance(value, Enum):
                data[f.name] = value.value
            else:
                data[f.name] = value
        return data

    @classmethod
    def from_dict(cls, data: dict[str, Any]) -> "Artifact":
        """Deserialize from a plain dict."""
        return cls(
            artifact_id=data["artifact_id"],
            artifact_type=ArtifactType(data["artifact_type"]),
            sender=data["sender"],
            receiver=data["receiver"],
            task_id=data["task_id"],
            trace_id=data["trace_id"],
            summary=data["summary"],
            payload=data["payload"],
            authority=Authority(data["authority"]),
            created_at=data["created_at"],
            parent_task_id=data.get("parent_task_id"),
            reply_to=data.get("reply_to"),
            resume_event=data.get("resume_event"),
        )


def create_artifact(
    *,
    artifact_type: ArtifactType,
    sender: str,
    receiver: str,
    task_id: str,
    trace_id: str,
    summary: str,
    payload: dict[str, Any],
    parent_task_id: str | None = None,
    reply_to: str | None = None,
    resume_event: str | None = None,
) -> Artifact:
    """Create an Artifact with authority=MEMBER (framework-enforced).

    Only ClusterRouter can create artifacts with authority=HUMAN.
    """
    return Artifact(
        artifact_id=str(uuid.uuid4()),
        artifact_type=artifact_type,
        sender=sender,
        receiver=receiver,
        task_id=task_id,
        trace_id=trace_id,
        summary=summary,
        payload=payload,
        authority=Authority.MEMBER,
        created_at=datetime.now(timezone.utc).isoformat(),
        parent_task_id=parent_task_id,
        reply_to=reply_to,
        resume_event=resume_event,
    )
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/core/test_artifact.py -v`
Expected: 6 passed

- [ ] **Step 5: Commit**

```bash
git add src/hal/core/artifact.py tests/core/test_artifact.py
git commit -m "feat: add immutable Artifact model with authority enforcement"
```

---

## Task 8: AgentDriver Protocol

**Files:**
- Create: `src/hal/core/driver.py`
- Create: `tests/core/test_driver.py`

- [ ] **Step 1: Write the failing tests**

`tests/core/test_driver.py`:
```python
from hal.core.driver import (
    AgentDriver,
    CancellationToken,
    DriverResult,
    DriverStatus,
    Lesson,
    LessonScope,
    SuspendContext,
)


def test_driver_result_completed():
    result = DriverResult(
        status=DriverStatus.COMPLETED,
        messages=[{"role": "assistant", "content": "Done"}],
        steps_taken=5,
        token_usage=1200,
        cost_usd=1.2,
        summary="Task completed successfully",
    )
    assert result.status == DriverStatus.COMPLETED
    assert result.steps_taken == 5
    assert result.token_usage == 1200
    assert result.cost_usd == 1.2
    assert result.lessons == []
    assert result.suspend_context is None


def test_driver_result_escalated():
    result = DriverResult(
        status=DriverStatus.ESCALATED,
        messages=[],
        steps_taken=3,
        token_usage=800,
        escalation_reason="Ambiguous requirement",
    )
    assert result.status == DriverStatus.ESCALATED
    assert result.escalation_reason == "Ambiguous requirement"


def test_driver_result_suspended_with_context():
    ctx = SuspendContext(
        resume_event_id="evt-123",
        confirmation_request=None,
    )
    result = DriverResult(
        status=DriverStatus.SUSPENDED,
        messages=[],
        steps_taken=2,
        token_usage=500,
        suspend_context=ctx,
    )
    assert result.suspend_context is not None
    assert result.suspend_context.resume_event_id == "evt-123"


def test_driver_result_with_lessons():
    lesson = Lesson(
        content="CocoaPods 1.15 incompatible with Xcode 16",
        trigger="build_failure",
        confidence=0.7,
        tags=["cocoapods", "xcode"],
        source_task="feat-login-sdk",
        scope=LessonScope.EPISODIC,
    )
    result = DriverResult(
        status=DriverStatus.COMPLETED,
        messages=[],
        steps_taken=10,
        token_usage=2000,
        cost_usd=2.0,
        lessons=[lesson],
    )
    assert len(result.lessons) == 1
    assert result.lessons[0].scope == LessonScope.EPISODIC


def test_driver_status_values():
    expected = {"COMPLETED", "ESCALATED", "SUSPENDED", "BUDGET_EXHAUSTED", "FAILED"}
    actual = {s.name for s in DriverStatus}
    assert actual == expected


def test_cancellation_token():
    token = CancellationToken()
    assert not token.is_cancelled
    token.cancel()
    assert token.is_cancelled


def test_agent_driver_is_protocol():
    """AgentDriver is a Protocol — any class with matching run() signature satisfies it."""

    class FakeDriver:
        async def run(
            self,
            *,
            messages: list[dict],
            tools: list,
            system_prompt: str,
            max_steps: int,
            cost_budget: float,
            cancellation_token: CancellationToken | None = None,
        ) -> DriverResult:
            return DriverResult(
                status=DriverStatus.COMPLETED,
                messages=messages,
                steps_taken=0,
                token_usage=0,
                cost_usd=0.0,
            )

    driver: AgentDriver = FakeDriver()  # type check — satisfies Protocol
    assert isinstance(driver, FakeDriver)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/core/test_driver.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`src/hal/core/driver.py`:
```python
"""AgentDriver protocol — the core abstraction for delegating agent execution.

HAL calls the Driver with context and constraints.
The Driver runs its internal LLM loop and returns a DriverResult.
"""

from dataclasses import dataclass, field
from enum import Enum
from typing import Any, Protocol, runtime_checkable


class DriverStatus(Enum):
    """Possible statuses returned by a Driver."""

    COMPLETED = "completed"
    ESCALATED = "escalated"
    SUSPENDED = "suspended"
    BUDGET_EXHAUSTED = "budget_exhausted"
    FAILED = "failed"


class LessonScope(Enum):
    """Where a Driver-reported lesson should be stored."""

    EPISODIC = "episodic"  # per-instance memory
    SHARED = "shared"  # cluster-wide memory (only effective from Leader Agent)


@dataclass
class Lesson:
    """A lesson extracted by the Driver during execution."""

    content: str
    trigger: str
    confidence: float
    tags: list[str]
    source_task: str | None = None
    scope: LessonScope = LessonScope.EPISODIC


@dataclass
class SuspendContext:
    """Context for a SUSPENDED DriverResult."""

    resume_event_id: str | None = None
    confirmation_request: dict[str, Any] | None = None


@dataclass
class DriverResult:
    """Result returned by AgentDriver.run()."""

    status: DriverStatus
    messages: list[dict[str, Any]]
    steps_taken: int
    token_usage: int
    cost_usd: float = 0.0
    summary: str | None = None
    escalation_reason: str | None = None
    output_artifacts: list[dict[str, Any]] = field(default_factory=list)
    lessons: list[Lesson] = field(default_factory=list)
    suspend_context: SuspendContext | None = None


class CancellationToken:
    """Lightweight async flag for mid-execution cancellation."""

    def __init__(self) -> None:
        self._cancelled = False

    @property
    def is_cancelled(self) -> bool:
        return self._cancelled

    def cancel(self) -> None:
        self._cancelled = True


@runtime_checkable
class AgentDriver(Protocol):
    """Protocol for agent execution drivers.

    HAL passes context and constraints; the Driver runs its LLM loop
    and returns a DriverResult.
    """

    async def run(
        self,
        *,
        messages: list[dict[str, Any]],
        tools: list[Any],
        system_prompt: str,
        max_steps: int,
        cost_budget: float,
        cancellation_token: CancellationToken | None = None,
    ) -> DriverResult: ...
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/core/test_driver.py -v`
Expected: 7 passed

- [ ] **Step 5: Commit**

```bash
git add src/hal/core/driver.py tests/core/test_driver.py
git commit -m "feat: add AgentDriver protocol and DriverResult data model"
```

---

## Task 9: HAL Bus Protocol + InMemoryBus

**Files:**
- Create: `src/hal/core/bus/protocol.py`
- Create: `src/hal/core/bus/memory.py`
- Create: `tests/core/bus/test_memory_bus.py`

- [ ] **Step 1: Write the failing tests**

`tests/core/bus/test_memory_bus.py`:
```python
import asyncio

import pytest

from hal.core.artifact import Artifact, ArtifactType, Authority, create_artifact
from hal.core.bus.memory import InMemoryBus


def _make_artifact(
    sender: str = "agent-a",
    receiver: str = "agent-b",
    task_id: str = "task-1",
    artifact_type: ArtifactType = ArtifactType.TASK_ASSIGNMENT,
    **kwargs,
) -> Artifact:
    return create_artifact(
        artifact_type=artifact_type,
        sender=sender,
        receiver=receiver,
        task_id=task_id,
        trace_id="trace-1",
        summary="test",
        payload={},
        **kwargs,
    )


@pytest.fixture
def bus() -> InMemoryBus:
    return InMemoryBus(max_inbox_size=5)


async def test_publish_and_subscribe(bus: InMemoryBus):
    received: list[Artifact] = []

    async def handler(artifact: Artifact) -> None:
        received.append(artifact)

    bus.subscribe("agent-b", handler)
    art = _make_artifact()
    result = await bus.publish(art)

    assert result is True
    assert len(received) == 1
    assert received[0].artifact_id == art.artifact_id


async def test_inbox_query(bus: InMemoryBus):
    bus.subscribe("agent-b", lambda a: asyncio.sleep(0))  # need subscriber for delivery

    art1 = _make_artifact(task_id="task-1")
    art2 = _make_artifact(task_id="task-2")
    await bus.publish(art1)
    await bus.publish(art2)

    all_items = bus.inbox_query("agent-b")
    assert len(all_items) == 2

    filtered = bus.inbox_query("agent-b", task_id="task-1")
    assert len(filtered) == 1
    assert filtered[0].task_id == "task-1"


async def test_inbox_empty(bus: InMemoryBus):
    assert bus.inbox_query("nonexistent") == []


async def test_backpressure_rejects_when_full(bus: InMemoryBus):
    bus.subscribe("agent-b", lambda a: asyncio.sleep(0))

    # Fill up inbox (max_inbox_size=5)
    for i in range(5):
        result = await bus.publish(_make_artifact(task_id=f"task-{i}"))
        assert result is True

    # 6th should be rejected
    result = await bus.publish(_make_artifact(task_id="task-overflow"))
    assert result is False


async def test_wait_for_reply(bus: InMemoryBus):
    request = _make_artifact(
        sender="agent-a",
        receiver="agent-b",
        artifact_type=ArtifactType.ESCALATION,
    )

    async def responder():
        await asyncio.sleep(0.05)
        reply = create_artifact(
            artifact_type=ArtifactType.DECISION,
            sender="agent-b",
            receiver="agent-a",
            task_id=request.task_id,
            trace_id=request.trace_id,
            summary="resolved",
            payload={"answer": 42},
            reply_to=request.artifact_id,
        )
        bus.subscribe("agent-a", lambda a: asyncio.sleep(0))
        await bus.publish(reply)

    asyncio.create_task(responder())
    reply = await bus.wait_for_reply(request, timeout=2.0)

    assert reply is not None
    assert reply.reply_to == request.artifact_id
    assert reply.payload["answer"] == 42


async def test_wait_for_reply_timeout(bus: InMemoryBus):
    request = _make_artifact()

    reply = await bus.wait_for_reply(request, timeout=0.1)
    assert reply is None


async def test_audit_trail_by_task(bus: InMemoryBus):
    bus.subscribe("agent-b", lambda a: asyncio.sleep(0))

    art1 = _make_artifact(task_id="task-x")
    art2 = _make_artifact(task_id="task-x")
    art3 = _make_artifact(task_id="task-y")
    await bus.publish(art1)
    await bus.publish(art2)
    await bus.publish(art3)

    trail = bus.audit_trail("task-x")
    assert len(trail) == 2
    assert all(a.task_id == "task-x" for a in trail)


async def test_audit_trail_by_trace_id(bus: InMemoryBus):
    bus.subscribe("agent-b", lambda a: asyncio.sleep(0))

    art1 = _make_artifact(task_id="task-1")
    art2 = _make_artifact(task_id="task-2")
    await bus.publish(art1)
    await bus.publish(art2)

    # Both share trace_id="trace-1" from _make_artifact default
    trail = bus.audit_trail(trace_id="trace-1")
    assert len(trail) == 2


async def test_idempotency_dedup(bus: InMemoryBus):
    received: list[Artifact] = []

    async def handler(artifact: Artifact) -> None:
        received.append(artifact)

    bus.subscribe("agent-b", handler)

    art = _make_artifact()
    await bus.publish(art)
    await bus.publish(art)  # same artifact_id — should be deduped

    assert len(received) == 1


async def test_interceptor_catches_matching_artifact(bus: InMemoryBus):
    intercepted: list[Artifact] = []
    normal: list[Artifact] = []

    async def intercept_handler(artifact: Artifact) -> None:
        intercepted.append(artifact)

    async def normal_handler(artifact: Artifact) -> None:
        normal.append(artifact)

    bus.add_interceptor(
        lambda a: a.artifact_type == ArtifactType.ESCALATION,
        intercept_handler,
    )
    bus.subscribe("agent-b", normal_handler)

    # Escalation → intercepted
    esc = _make_artifact(artifact_type=ArtifactType.ESCALATION)
    await bus.publish(esc)

    # Decision → normal delivery
    dec = _make_artifact(artifact_type=ArtifactType.DECISION)
    await bus.publish(dec)

    assert len(intercepted) == 1
    assert intercepted[0].artifact_type == ArtifactType.ESCALATION
    assert len(normal) == 1
    assert normal[0].artifact_type == ArtifactType.DECISION


async def test_intercepted_artifact_not_in_inbox(bus: InMemoryBus):
    async def intercept_handler(artifact: Artifact) -> None:
        pass  # swallow

    bus.add_interceptor(
        lambda a: a.artifact_type == ArtifactType.CONFIRMATION_REQUEST,
        intercept_handler,
    )

    art = _make_artifact(artifact_type=ArtifactType.CONFIRMATION_REQUEST)
    result = await bus.publish(art)

    assert result is True
    assert bus.inbox_query("agent-b") == []  # not delivered to inbox


async def test_publish_without_subscriber():
    bus = InMemoryBus(max_inbox_size=10)
    art = _make_artifact()
    # No subscriber — artifact stored in inbox but no handler called
    result = await bus.publish(art)
    assert result is True
    items = bus.inbox_query("agent-b")
    assert len(items) == 1
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/core/bus/test_memory_bus.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write Bus Protocol**

`src/hal/core/bus/protocol.py`:
```python
"""HAL Bus protocol — async artifact transport between agent instances."""

from typing import Any, Protocol, runtime_checkable
from collections.abc import Awaitable, Callable

from hal.core.artifact import Artifact

ArtifactHandler = Callable[[Artifact], Awaitable[None]]


@runtime_checkable
class Bus(Protocol):
    """Protocol for HAL Bus backends.

    All agent communication goes through the Bus interface.
    Implementations: InMemoryBus (V1), SQLiteBus (V2), RedisBus (V2).
    """

    async def publish(self, artifact: Artifact) -> bool:
        """Publish an artifact to the target agent's inbox.

        Returns True if delivered, False if rejected (backpressure).
        """
        ...

    def subscribe(self, instance_id: str, handler: ArtifactHandler) -> None:
        """Register a handler for artifacts addressed to instance_id."""
        ...

    async def wait_for_reply(
        self, request: Artifact, *, timeout: float
    ) -> Artifact | None:
        """Publish request and wait for a reply artifact (matching reply_to).

        Returns None on timeout.
        """
        ...

    def inbox_query(
        self, instance_id: str, *, task_id: str | None = None
    ) -> list[Artifact]:
        """Retrieve pending artifacts for an instance, optionally filtered by task."""
        ...

    def audit_trail(
        self, task_id: str | None = None, *, trace_id: str | None = None
    ) -> list[Artifact]:
        """Retrieve all artifacts for a given task and/or trace_id."""
        ...

    def add_interceptor(
        self,
        predicate: Callable[[Artifact], bool],
        handler: ArtifactHandler,
    ) -> None:
        """Register an interceptor. Matching artifacts are routed to handler instead of normal delivery."""
        ...
```

- [ ] **Step 4: Write InMemoryBus implementation**

`src/hal/core/bus/memory.py`:
```python
"""InMemoryBus — single-process async Bus implementation."""

import asyncio
from collections import defaultdict
from collections.abc import Callable

from hal.core.artifact import Artifact
from hal.core.bus.protocol import ArtifactHandler


class InMemoryBus:
    """In-process async Bus for single-process deployment.

    Features:
    - Per-instance inbox with configurable max size (backpressure)
    - Artifact deduplication by artifact_id (V1: in-memory set, not persisted)
    - Full audit trail by task_id
    - Wait-for-reply with timeout
    """

    def __init__(self, max_inbox_size: int = 100) -> None:
        self._max_inbox_size = max_inbox_size
        self._handlers: dict[str, list[ArtifactHandler]] = defaultdict(list)
        self._inboxes: dict[str, list[Artifact]] = defaultdict(list)
        self._seen_ids: set[str] = set()  # V1 dedup — not persisted
        self._all_artifacts: list[Artifact] = []  # audit trail
        self._reply_waiters: dict[str, asyncio.Future[Artifact]] = {}
        self._interceptors: list[tuple[Callable[[Artifact], bool], ArtifactHandler]] = []

    async def publish(self, artifact: Artifact) -> bool:
        """Publish artifact to target inbox. Returns False if inbox full."""
        # Dedup check
        if artifact.artifact_id in self._seen_ids:
            return True  # already processed, silently accept
        self._seen_ids.add(artifact.artifact_id)

        # Check interceptors — matching artifacts bypass normal delivery
        for predicate, int_handler in self._interceptors:
            if predicate(artifact):
                self._all_artifacts.append(artifact)  # still audit
                await int_handler(artifact)
                return True  # intercepted

        receiver = artifact.receiver

        # Backpressure
        if len(self._inboxes[receiver]) >= self._max_inbox_size:
            return False

        # Store in inbox and audit trail
        self._inboxes[receiver].append(artifact)
        self._all_artifacts.append(artifact)

        # Check if this is a reply someone is waiting for
        if artifact.reply_to and artifact.reply_to in self._reply_waiters:
            future = self._reply_waiters.pop(artifact.reply_to)
            if not future.done():
                future.set_result(artifact)

        # Deliver to handlers
        for handler in self._handlers.get(receiver, []):
            await handler(artifact)

        return True

    def subscribe(self, instance_id: str, handler: ArtifactHandler) -> None:
        """Register a handler for artifacts addressed to instance_id."""
        self._handlers[instance_id].append(handler)

    async def wait_for_reply(
        self, request: Artifact, *, timeout: float
    ) -> Artifact | None:
        """Publish request and wait for a reply (matching reply_to=request.artifact_id)."""
        loop = asyncio.get_running_loop()
        future: asyncio.Future[Artifact] = loop.create_future()
        self._reply_waiters[request.artifact_id] = future

        # Publish the request first
        published = await self.publish(request)
        if not published:
            self._reply_waiters.pop(request.artifact_id, None)
            return None

        try:
            return await asyncio.wait_for(future, timeout=timeout)
        except asyncio.TimeoutError:
            self._reply_waiters.pop(request.artifact_id, None)
            return None

    def inbox_query(
        self, instance_id: str, *, task_id: str | None = None
    ) -> list[Artifact]:
        """Retrieve artifacts in an instance's inbox, optionally filtered by task_id."""
        items = self._inboxes.get(instance_id, [])
        if task_id is not None:
            return [a for a in items if a.task_id == task_id]
        return list(items)

    def audit_trail(
        self, task_id: str | None = None, *, trace_id: str | None = None
    ) -> list[Artifact]:
        """Retrieve all artifacts for a given task and/or trace_id."""
        results = self._all_artifacts
        if task_id is not None:
            results = [a for a in results if a.task_id == task_id]
        if trace_id is not None:
            results = [a for a in results if a.trace_id == trace_id]
        return results

    def add_interceptor(
        self,
        predicate: Callable[[Artifact], bool],
        handler: ArtifactHandler,
    ) -> None:
        """Register an interceptor for matching artifacts."""
        self._interceptors.append((predicate, handler))
```

- [ ] **Step 5: Update `src/hal/core/bus/__init__.py`**

```python
"""HAL Bus — async artifact transport between agent instances."""

from hal.core.bus.protocol import Bus, ArtifactHandler
from hal.core.bus.memory import InMemoryBus

__all__ = ["Bus", "ArtifactHandler", "InMemoryBus"]
```

- [ ] **Step 6: Run test to verify it passes**

Run: `pytest tests/core/bus/test_memory_bus.py -v`
Expected: 10 passed

- [ ] **Step 7: Commit**

```bash
git add src/hal/core/bus/ tests/core/bus/
git commit -m "feat: add HAL Bus protocol and InMemoryBus with backpressure and dedup"
```

---

## Task 10: Integration Test

**Files:**
- Create: `tests/core/test_integration.py`

- [ ] **Step 1: Write the integration test**

`tests/core/test_integration.py`:
```python
"""Integration test — verify Core Kernel components work together.

Scenario: Publish a task_assignment Artifact through InMemoryBus,
verify AsyncEventBus emits the event, and HALLogger produces
traceable JSON output.
"""

import asyncio
import json

import pytest

from hal.core.artifact import ArtifactType, create_artifact
from hal.core.bus.memory import InMemoryBus
from hal.core.event_bus import AsyncEventBus
from hal.core.logger import HALLogger, configure_logging
from hal.core.plugin_registry import PluginRegistry


async def test_full_artifact_lifecycle():
    """Publish artifact → event bus emits → logger traces — all with same trace_id."""
    # --- Setup ---
    event_bus = AsyncEventBus()
    hal_bus = InMemoryBus(max_inbox_size=10)
    registry = PluginRegistry()
    registry.register("transport", "inmemory", hal_bus)
    logger = HALLogger(agent_id="ios-dev-01")

    # Track events
    emitted_events: list[dict] = []

    async def on_artifact_published(data: dict) -> None:
        emitted_events.append(data)

    event_bus.subscribe("artifact.published", on_artifact_published)

    # --- Create and publish artifact ---
    trace_id = "trace-integration-001"
    artifact = create_artifact(
        artifact_type=ArtifactType.TASK_ASSIGNMENT,
        sender="pm-01",
        receiver="ios-dev-01",
        task_id="task-int-1",
        trace_id=trace_id,
        summary="Implement login flow",
        payload={"description": "OAuth2 login for iOS SDK"},
        parent_task_id="main-task-1",
    )

    # Simulate what AgentService would do: publish to Bus + emit event
    received: list = []
    hal_bus.subscribe("ios-dev-01", lambda a: received.append(a) or asyncio.sleep(0))

    result = await hal_bus.publish(artifact)
    assert result is True

    await event_bus.emit("artifact.published", {
        "artifact_id": artifact.artifact_id,
        "type": artifact.artifact_type.value,
        "sender": artifact.sender,
        "receiver": artifact.receiver,
        "trace_id": artifact.trace_id,
    })

    # --- Verify ---
    # Bus delivered the artifact
    assert len(received) == 1
    assert received[0].artifact_id == artifact.artifact_id

    # Event bus captured the event
    assert len(emitted_events) == 1
    assert emitted_events[0]["trace_id"] == trace_id
    assert emitted_events[0]["type"] == "task_assignment"

    # Inbox query works
    inbox = hal_bus.inbox_query("ios-dev-01")
    assert len(inbox) == 1

    # Audit trail works
    trail = hal_bus.audit_trail("task-int-1")
    assert len(trail) == 1

    # Plugin registry has the bus
    assert registry.get("transport", "inmemory") is hal_bus

    # Artifact is immutable
    with pytest.raises(AttributeError):
        artifact.summary = "hacked"  # type: ignore[misc]

    # Authority is MEMBER (framework-enforced)
    assert artifact.authority.value == "member"

    # Serialization round-trip preserves all fields
    from hal.core.artifact import Artifact

    restored = Artifact.from_dict(artifact.to_dict())
    assert restored.parent_task_id == "main-task-1"
    assert restored.trace_id == trace_id


async def test_bus_with_event_bus_backpressure():
    """When bus inbox is full, publish returns False — event bus still works."""
    event_bus = AsyncEventBus()
    hal_bus = InMemoryBus(max_inbox_size=2)

    rejection_events: list[dict] = []

    async def on_rejection(data: dict) -> None:
        rejection_events.append(data)

    event_bus.subscribe("bus.publish_rejected", on_rejection)

    hal_bus.subscribe("target", lambda a: asyncio.sleep(0))

    # Fill inbox
    for i in range(2):
        art = create_artifact(
            artifact_type=ArtifactType.DECISION,
            sender="a",
            receiver="target",
            task_id=f"t-{i}",
            trace_id="tr",
            summary="x",
            payload={},
        )
        assert await hal_bus.publish(art) is True

    # 3rd rejected
    overflow = create_artifact(
        artifact_type=ArtifactType.DECISION,
        sender="a",
        receiver="target",
        task_id="t-overflow",
        trace_id="tr",
        summary="x",
        payload={},
    )
    rejected = await hal_bus.publish(overflow)
    assert rejected is False

    # Simulate emitting rejection event (as framework would)
    await event_bus.emit("bus.publish_rejected", {
        "artifact_id": overflow.artifact_id,
        "receiver": "target",
        "reason": "inbox_full",
    })

    assert len(rejection_events) == 1
    assert rejection_events[0]["reason"] == "inbox_full"
```

- [ ] **Step 2: Run integration test**

Run: `pytest tests/core/test_integration.py -v`
Expected: 2 passed

- [ ] **Step 3: Run full test suite**

Run: `pytest tests/ -v`
Expected: All tests pass (approx. 38 tests total)

- [ ] **Step 4: Commit**

```bash
git add tests/core/test_integration.py
git commit -m "test: add Core Kernel integration tests"
```

---

## Summary

| Task | Component | Test count (est.) |
|------|-----------|-------------------|
| 1 | Project scaffolding | 0 |
| 2 | Error classification | 4 |
| 3 | Config Loader | 8 |
| 4 | AsyncEventBus | 6 |
| 5 | Structured Logger | 3 |
| 6 | Plugin Registry | 6 |
| 7 | Artifact Model | 6 |
| 8 | AgentDriver Protocol | 7 |
| 9 | HAL Bus + InMemoryBus | 13 |
| 10 | Integration | 2 |
| **Total** | | **~55** |
