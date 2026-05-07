# Plan 2: Tool Arsenal — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build Layer 2 — the tool protocol, registry, loader (three-tier config), and SandboxedTool wrapper with visibility control and risk-level enforcement.

**Architecture:** Tools implement a unified protocol (execute/validate/rollback). The ToolRegistry stores tool definitions by name. The ToolLoader resolves three-tier config (preset toolkit → additive/subtractive → custom) into a set of tools for a role. SandboxedTool wraps each tool transparently, enforcing validation, risk-level confirmation (PENDING for high-risk), and audit events.

**Tech Stack:** Python 3.9+, asyncio, pytest, pytest-asyncio

**Spec:** `docs/superpowers/specs/2026-04-27-hal-design.md` — Layer 2: Tool Arsenal

**Depends on:** Plan 1 (Core Kernel — AsyncEventBus, PluginRegistry, errors)

---

## File Structure

```
src/hal/tools/
├── __init__.py
├── protocol.py        # Tool protocol, ToolResult, ToolStatus, RiskLevel
├── registry.py        # ToolRegistry — register/lookup tools and toolkit presets
├── loader.py          # ToolLoader — resolve three-tier config → tool set
└── sandbox.py         # SandboxedTool — validation + risk check + execute + audit

tests/tools/
├── __init__.py
├── test_protocol.py
├── test_registry.py
├── test_loader.py
├── test_sandbox.py
└── test_tools_integration.py
```

---

## Task 1: Tool Protocol

**Files:**
- Create: `src/hal/tools/__init__.py`
- Create: `src/hal/tools/protocol.py`
- Create: `tests/tools/__init__.py`
- Create: `tests/tools/test_protocol.py`

- [ ] **Step 1: Write the failing tests**

`tests/tools/test_protocol.py`:
```python
import pytest

from hal.tools.protocol import RiskLevel, ToolResult, ToolStatus


def test_tool_result_success():
    result = ToolResult(status=ToolStatus.SUCCESS, data={"output": "hello"})
    assert result.status == ToolStatus.SUCCESS
    assert result.data == {"output": "hello"}
    assert result.resume_event_id is None


def test_tool_result_failure():
    result = ToolResult(status=ToolStatus.FAILURE, data={"error": "not found"})
    assert result.status == ToolStatus.FAILURE


def test_tool_result_pending():
    result = ToolResult(
        status=ToolStatus.PENDING,
        data={"message": "waiting for CI"},
        resume_event_id="evt-ci-123",
    )
    assert result.status == ToolStatus.PENDING
    assert result.resume_event_id == "evt-ci-123"


def test_risk_levels():
    assert RiskLevel.LOW.value == "low"
    assert RiskLevel.MEDIUM.value == "medium"
    assert RiskLevel.HIGH.value == "high"


def test_tool_status_values():
    expected = {"SUCCESS", "FAILURE", "PENDING"}
    assert {s.name for s in ToolStatus} == expected
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/tools/test_protocol.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`src/hal/tools/__init__.py`:
```python
"""HAL Tool Arsenal — role-based dynamic tool loading with visibility control."""
```

`src/hal/tools/protocol.py`:
```python
"""Tool protocol — unified interface for all HAL tools.

Every tool implements execute/validate/rollback and declares a risk_level.
Tool results carry one of three statuses: SUCCESS, FAILURE, PENDING.
"""

from dataclasses import dataclass, field
from enum import Enum
from typing import Any, Protocol, runtime_checkable


class ToolStatus(Enum):
    """Result status from a tool execution."""

    SUCCESS = "success"
    FAILURE = "failure"
    PENDING = "pending"


class RiskLevel(Enum):
    """Tool risk classification.

    LOW/MEDIUM: informational (logging/audit only).
    HIGH: behavioral — triggers Human confirmation before execution.
    """

    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"


@dataclass
class ToolResult:
    """Result of a tool execution."""

    status: ToolStatus
    data: dict[str, Any] = field(default_factory=dict)
    resume_event_id: str | None = None


@runtime_checkable
class Tool(Protocol):
    """Protocol for HAL tools.

    Every tool declares a name, description, risk_level, and parameter schema.
    """

    @property
    def name(self) -> str: ...

    @property
    def description(self) -> str: ...

    @property
    def risk_level(self) -> RiskLevel: ...

    @property
    def parameters_schema(self) -> dict[str, Any]:
        """JSON Schema for tool parameters (used by LLM for function calling)."""
        ...

    def validate(self, params: dict[str, Any]) -> None:
        """Pre-check parameters. Raises ValueError if invalid."""
        ...

    async def execute(self, params: dict[str, Any]) -> ToolResult:
        """Perform the action. Async — may involve IO."""
        ...

    async def rollback(self, params: dict[str, Any]) -> None:
        """Undo the action if possible. Optional — default is no-op."""
        ...
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/tools/test_protocol.py -v`
Expected: 5 passed

- [ ] **Step 5: Commit**

```bash
git add src/hal/tools/ tests/tools/
git commit -m "feat: add Tool protocol with ToolResult and RiskLevel"
```

---

## Task 2: Tool Registry

**Files:**
- Create: `src/hal/tools/registry.py`
- Create: `tests/tools/test_registry.py`

- [ ] **Step 1: Write the failing tests**

`tests/tools/test_registry.py`:
```python
import pytest

from hal.tools.protocol import RiskLevel, Tool, ToolResult, ToolStatus
from hal.tools.registry import ToolRegistry


class FakeTool:
    """Minimal Tool implementation for testing."""

    def __init__(self, name: str, risk: RiskLevel = RiskLevel.LOW) -> None:
        self._name = name
        self._risk = risk

    @property
    def name(self) -> str:
        return self._name

    @property
    def description(self) -> str:
        return f"Fake tool: {self._name}"

    @property
    def risk_level(self) -> RiskLevel:
        return self._risk

    @property
    def parameters_schema(self) -> dict:
        return {"type": "object", "properties": {}}

    def validate(self, params: dict) -> None:
        pass

    async def execute(self, params: dict) -> ToolResult:
        return ToolResult(status=ToolStatus.SUCCESS)

    async def rollback(self, params: dict) -> None:
        pass


@pytest.fixture
def registry() -> ToolRegistry:
    return ToolRegistry()


def test_register_and_get_tool(registry: ToolRegistry):
    tool = FakeTool("git")
    registry.register_tool(tool)
    assert registry.get_tool("git") is tool


def test_get_nonexistent_tool(registry: ToolRegistry):
    assert registry.get_tool("nonexistent") is None


def test_list_tools(registry: ToolRegistry):
    registry.register_tool(FakeTool("git"))
    registry.register_tool(FakeTool("filesystem"))
    registry.register_tool(FakeTool("build"))

    names = registry.list_tools()
    assert sorted(names) == ["build", "filesystem", "git"]


def test_duplicate_tool_raises(registry: ToolRegistry):
    registry.register_tool(FakeTool("git"))
    with pytest.raises(ValueError, match="already registered"):
        registry.register_tool(FakeTool("git"))


def test_register_toolkit_preset(registry: ToolRegistry):
    registry.register_tool(FakeTool("git"))
    registry.register_tool(FakeTool("filesystem"))
    registry.register_tool(FakeTool("build"))
    registry.register_tool(FakeTool("search"))

    registry.register_toolkit("mobile_dev", ["git", "filesystem", "build", "search"])
    assert sorted(registry.get_toolkit("mobile_dev")) == [
        "build", "filesystem", "git", "search",
    ]


def test_get_nonexistent_toolkit(registry: ToolRegistry):
    assert registry.get_toolkit("nonexistent") is None


def test_register_toolkit_with_unknown_tool_raises(registry: ToolRegistry):
    registry.register_tool(FakeTool("git"))
    with pytest.raises(ValueError, match="not registered"):
        registry.register_toolkit("bad_kit", ["git", "nonexistent_tool"])
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/tools/test_registry.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`src/hal/tools/registry.py`:
```python
"""Tool registry — register tools by name and define toolkit presets."""

from hal.tools.protocol import Tool


class ToolRegistry:
    """Central registry for tools and toolkit presets.

    Tools are registered individually by name.
    Toolkit presets are named bundles of tool names for three-tier config.
    """

    def __init__(self) -> None:
        self._tools: dict[str, Tool] = {}
        self._toolkits: dict[str, list[str]] = {}

    def register_tool(self, tool: Tool) -> None:
        """Register a tool by its name. Raises ValueError if already registered."""
        if tool.name in self._tools:
            raise ValueError(f"Tool '{tool.name}' already registered")
        self._tools[tool.name] = tool

    def get_tool(self, name: str) -> Tool | None:
        """Get a tool by name. Returns None if not found."""
        return self._tools.get(name)

    def list_tools(self) -> list[str]:
        """List all registered tool names."""
        return list(self._tools.keys())

    def register_toolkit(self, name: str, tool_names: list[str]) -> None:
        """Register a toolkit preset — a named bundle of tool names.

        All tool names must already be registered. Raises ValueError otherwise.
        """
        for tool_name in tool_names:
            if tool_name not in self._tools:
                raise ValueError(
                    f"Tool '{tool_name}' not registered — "
                    f"cannot include in toolkit '{name}'"
                )
        self._toolkits[name] = list(tool_names)

    def get_toolkit(self, name: str) -> list[str] | None:
        """Get toolkit preset tool names. Returns None if not found."""
        return self._toolkits.get(name)
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/tools/test_registry.py -v`
Expected: 7 passed

- [ ] **Step 5: Commit**

```bash
git add src/hal/tools/registry.py tests/tools/test_registry.py
git commit -m "feat: add ToolRegistry with toolkit preset support"
```

---

## Task 3: Tool Loader (Three-Tier Config)

**Files:**
- Create: `src/hal/tools/loader.py`
- Create: `tests/tools/test_loader.py`

- [ ] **Step 1: Write the failing tests**

`tests/tools/test_loader.py`:
```python
import pytest

from hal.tools.loader import ToolLoader
from hal.tools.protocol import RiskLevel, ToolResult, ToolStatus
from hal.tools.registry import ToolRegistry


class FakeTool:
    def __init__(self, name: str) -> None:
        self._name = name

    @property
    def name(self) -> str:
        return self._name

    @property
    def description(self) -> str:
        return f"Fake: {self._name}"

    @property
    def risk_level(self) -> RiskLevel:
        return RiskLevel.LOW

    @property
    def parameters_schema(self) -> dict:
        return {}

    def validate(self, params: dict) -> None:
        pass

    async def execute(self, params: dict) -> ToolResult:
        return ToolResult(status=ToolStatus.SUCCESS)

    async def rollback(self, params: dict) -> None:
        pass


@pytest.fixture
def registry() -> ToolRegistry:
    reg = ToolRegistry()
    for name in ["git", "filesystem", "build", "search", "deploy", "cocoapods"]:
        reg.register_tool(FakeTool(name))
    reg.register_toolkit("mobile_dev", ["git", "filesystem", "build", "search"])
    return reg


def test_tier1_toolkit_only(registry: ToolRegistry):
    """Tier 1: just a toolkit preset name."""
    config = {"toolkit": "mobile_dev"}
    loader = ToolLoader(registry)
    tools = loader.load(config)

    names = sorted(t.name for t in tools)
    assert names == ["build", "filesystem", "git", "search"]


def test_tier2_additive(registry: ToolRegistry):
    """Tier 2: toolkit + tools_add."""
    config = {"toolkit": "mobile_dev", "tools_add": ["cocoapods"]}
    loader = ToolLoader(registry)
    tools = loader.load(config)

    names = sorted(t.name for t in tools)
    assert names == ["build", "cocoapods", "filesystem", "git", "search"]


def test_tier2_subtractive(registry: ToolRegistry):
    """Tier 2: toolkit + tools_remove."""
    config = {"toolkit": "mobile_dev", "tools_remove": ["deploy"]}
    loader = ToolLoader(registry)
    tools = loader.load(config)

    # deploy was not in mobile_dev, so no change
    names = sorted(t.name for t in tools)
    assert names == ["build", "filesystem", "git", "search"]


def test_tier2_add_and_remove(registry: ToolRegistry):
    """Tier 2: toolkit + tools_add + tools_remove."""
    config = {
        "toolkit": "mobile_dev",
        "tools_add": ["cocoapods"],
        "tools_remove": ["build"],
    }
    loader = ToolLoader(registry)
    tools = loader.load(config)

    names = sorted(t.name for t in tools)
    assert names == ["cocoapods", "filesystem", "git", "search"]


def test_no_toolkit_no_tools_returns_empty(registry: ToolRegistry):
    """No toolkit and no tools → empty set."""
    config = {}
    loader = ToolLoader(registry)
    tools = loader.load(config)
    assert tools == []


def test_unknown_toolkit_raises(registry: ToolRegistry):
    config = {"toolkit": "nonexistent"}
    loader = ToolLoader(registry)
    with pytest.raises(ValueError, match="Unknown toolkit"):
        loader.load(config)


def test_unknown_tool_in_add_raises(registry: ToolRegistry):
    config = {"toolkit": "mobile_dev", "tools_add": ["unknown_tool"]}
    loader = ToolLoader(registry)
    with pytest.raises(ValueError, match="not found"):
        loader.load(config)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/tools/test_loader.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`src/hal/tools/loader.py`:
```python
"""Tool loader — resolve three-tier tool configuration to a concrete tool set.

Tier 1: toolkit preset name only.
Tier 2: toolkit + tools_add / tools_remove.
Tier 3: fully custom (V2+ — not implemented).
"""

from typing import Any

from hal.tools.protocol import Tool
from hal.tools.registry import ToolRegistry


class ToolLoader:
    """Resolves role tool configuration into a list of Tool instances."""

    def __init__(self, registry: ToolRegistry) -> None:
        self._registry = registry

    def load(self, tools_config: dict[str, Any]) -> list[Tool]:
        """Load tools based on three-tier config.

        Args:
            tools_config: Dict with optional keys: toolkit, tools_add, tools_remove.

        Returns:
            List of Tool instances for the role.
        """
        toolkit_name = tools_config.get("toolkit")
        tools_add = tools_config.get("tools_add", [])
        tools_remove = tools_config.get("tools_remove", [])

        # Start with toolkit preset or empty set
        if toolkit_name:
            preset = self._registry.get_toolkit(toolkit_name)
            if preset is None:
                raise ValueError(f"Unknown toolkit: '{toolkit_name}'")
            tool_names = set(preset)
        else:
            tool_names = set()

        # Add
        for name in tools_add:
            if self._registry.get_tool(name) is None:
                raise ValueError(f"Tool '{name}' not found in registry")
            tool_names.add(name)

        # Remove
        tool_names -= set(tools_remove)

        # Resolve to Tool instances
        tools: list[Tool] = []
        for name in sorted(tool_names):  # sorted for deterministic order
            tool = self._registry.get_tool(name)
            assert tool is not None  # guaranteed by registry/add checks
            tools.append(tool)

        return tools
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/tools/test_loader.py -v`
Expected: 7 passed

- [ ] **Step 5: Commit**

```bash
git add src/hal/tools/loader.py tests/tools/test_loader.py
git commit -m "feat: add ToolLoader with three-tier config resolution"
```

---

## Task 4: SandboxedTool

**Files:**
- Create: `src/hal/tools/sandbox.py`
- Create: `tests/tools/test_sandbox.py`

- [ ] **Step 1: Write the failing tests**

`tests/tools/test_sandbox.py`:
```python
import pytest

from hal.core.event_bus import AsyncEventBus
from hal.tools.protocol import RiskLevel, ToolResult, ToolStatus
from hal.tools.sandbox import SandboxedTool


class FakeTool:
    def __init__(
        self,
        name: str = "test_tool",
        risk: RiskLevel = RiskLevel.LOW,
        validate_error: str | None = None,
        execute_result: ToolResult | None = None,
    ) -> None:
        self._name = name
        self._risk = risk
        self._validate_error = validate_error
        self._execute_result = execute_result or ToolResult(
            status=ToolStatus.SUCCESS, data={"result": "ok"}
        )
        self.execute_called = False
        self.rollback_called = False

    @property
    def name(self) -> str:
        return self._name

    @property
    def description(self) -> str:
        return f"Fake: {self._name}"

    @property
    def risk_level(self) -> RiskLevel:
        return self._risk

    @property
    def parameters_schema(self) -> dict:
        return {"type": "object"}

    def validate(self, params: dict) -> None:
        if self._validate_error:
            raise ValueError(self._validate_error)

    async def execute(self, params: dict) -> ToolResult:
        self.execute_called = True
        return self._execute_result

    async def rollback(self, params: dict) -> None:
        self.rollback_called = True


@pytest.fixture
def event_bus() -> AsyncEventBus:
    return AsyncEventBus()


async def test_low_risk_executes_normally(event_bus: AsyncEventBus):
    inner = FakeTool(risk=RiskLevel.LOW)
    sandboxed = SandboxedTool(inner, event_bus)

    result = await sandboxed.execute({"key": "value"})

    assert result.status == ToolStatus.SUCCESS
    assert inner.execute_called


async def test_medium_risk_executes_normally(event_bus: AsyncEventBus):
    inner = FakeTool(risk=RiskLevel.MEDIUM)
    sandboxed = SandboxedTool(inner, event_bus)

    result = await sandboxed.execute({})
    assert result.status == ToolStatus.SUCCESS
    assert inner.execute_called


async def test_high_risk_returns_pending(event_bus: AsyncEventBus):
    inner = FakeTool(risk=RiskLevel.HIGH)
    sandboxed = SandboxedTool(inner, event_bus)

    result = await sandboxed.execute({"action": "deploy"})

    assert result.status == ToolStatus.PENDING
    assert result.data.get("confirmation_request") is not None
    assert result.data["confirmation_request"]["tool_name"] == "test_tool"
    assert result.data["confirmation_request"]["params"] == {"action": "deploy"}
    assert not inner.execute_called  # NOT executed yet


async def test_validation_failure_returns_failure(event_bus: AsyncEventBus):
    inner = FakeTool(validate_error="bad params")
    sandboxed = SandboxedTool(inner, event_bus)

    result = await sandboxed.execute({"bad": "data"})

    assert result.status == ToolStatus.FAILURE
    assert "bad params" in result.data["error"]
    assert not inner.execute_called


async def test_audit_event_emitted(event_bus: AsyncEventBus):
    events: list[dict] = []

    async def handler(data: dict) -> None:
        events.append(data)

    event_bus.subscribe("tool.called", handler)

    inner = FakeTool(name="git")
    sandboxed = SandboxedTool(inner, event_bus)
    await sandboxed.execute({"action": "commit"})

    assert len(events) == 1
    assert events[0]["tool_name"] == "git"
    assert events[0]["params"] == {"action": "commit"}
    assert events[0]["result_status"] == "success"


async def test_audit_event_on_high_risk_pending(event_bus: AsyncEventBus):
    events: list[dict] = []

    async def handler(data: dict) -> None:
        events.append(data)

    event_bus.subscribe("tool.called", handler)

    inner = FakeTool(name="deploy", risk=RiskLevel.HIGH)
    sandboxed = SandboxedTool(inner, event_bus)
    await sandboxed.execute({})

    assert len(events) == 1
    assert events[0]["result_status"] == "pending"


async def test_sandboxed_tool_exposes_inner_properties(event_bus: AsyncEventBus):
    inner = FakeTool(name="git", risk=RiskLevel.LOW)
    sandboxed = SandboxedTool(inner, event_bus)

    assert sandboxed.name == "git"
    assert sandboxed.description == "Fake: git"
    assert sandboxed.risk_level == RiskLevel.LOW
    assert sandboxed.parameters_schema == {"type": "object"}


async def test_rollback_delegates_to_inner(event_bus: AsyncEventBus):
    inner = FakeTool()
    sandboxed = SandboxedTool(inner, event_bus)
    await sandboxed.rollback({"key": "value"})
    assert inner.rollback_called


async def test_execution_error_returns_failure(event_bus: AsyncEventBus):
    class ErrorTool(FakeTool):
        async def execute(self, params: dict) -> ToolResult:
            raise RuntimeError("disk full")

    inner = ErrorTool()
    sandboxed = SandboxedTool(inner, event_bus)
    result = await sandboxed.execute({})

    assert result.status == ToolStatus.FAILURE
    assert "disk full" in result.data["error"]
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/tools/test_sandbox.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`src/hal/tools/sandbox.py`:
```python
"""SandboxedTool — transparent wrapper enforcing validation, risk checks, and audit.

The Driver receives SandboxedTool instances instead of raw Tools.
The wrapper follows the same interface, so the Driver is unaware of wrapping.
"""

import logging
from typing import Any

from hal.core.event_bus import AsyncEventBus
from hal.tools.protocol import RiskLevel, Tool, ToolResult, ToolStatus

logger = logging.getLogger(__name__)


class SandboxedTool:
    """Wraps a Tool with validation → risk check → execute → audit pipeline.

    Steps on each execute():
    1. Validate — call inner tool's validate()
    2. Risk check — if HIGH, return PENDING with confirmation_request
    3. Execute — call inner tool's execute()
    4. Audit — emit tool.called event
    """

    def __init__(self, tool: Tool, event_bus: AsyncEventBus) -> None:
        self._tool = tool
        self._event_bus = event_bus

    @property
    def name(self) -> str:
        return self._tool.name

    @property
    def description(self) -> str:
        return self._tool.description

    @property
    def risk_level(self) -> RiskLevel:
        return self._tool.risk_level

    @property
    def parameters_schema(self) -> dict[str, Any]:
        return self._tool.parameters_schema

    def validate(self, params: dict[str, Any]) -> None:
        self._tool.validate(params)

    async def execute(self, params: dict[str, Any]) -> ToolResult:
        """Execute with validation → risk check → execute → audit pipeline."""
        # 1. Validate
        try:
            self._tool.validate(params)
        except (ValueError, TypeError) as e:
            result = ToolResult(status=ToolStatus.FAILURE, data={"error": str(e)})
            await self._emit_audit(params, result)
            return result

        # 2. Risk check
        if self._tool.risk_level == RiskLevel.HIGH:
            result = ToolResult(
                status=ToolStatus.PENDING,
                data={
                    "confirmation_request": {
                        "tool_name": self._tool.name,
                        "params": params,
                        "risk_level": "high",
                        "description": self._tool.description,
                    }
                },
            )
            await self._emit_audit(params, result)
            return result

        # 3. Execute
        try:
            result = await self._tool.execute(params)
        except Exception as e:
            logger.exception("Tool '%s' execution failed", self._tool.name)
            result = ToolResult(status=ToolStatus.FAILURE, data={"error": str(e)})

        # 4. Audit
        await self._emit_audit(params, result)
        return result

    async def rollback(self, params: dict[str, Any]) -> None:
        await self._tool.rollback(params)

    async def _emit_audit(self, params: dict[str, Any], result: ToolResult) -> None:
        await self._event_bus.emit("tool.called", {
            "tool_name": self._tool.name,
            "params": params,
            "result_status": result.status.value,
        })
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/tools/test_sandbox.py -v`
Expected: 9 passed

- [ ] **Step 5: Commit**

```bash
git add src/hal/tools/sandbox.py tests/tools/test_sandbox.py
git commit -m "feat: add SandboxedTool with validation, risk check, and audit"
```

---

## Task 5: Integration Test

**Files:**
- Create: `tests/tools/test_tools_integration.py`

- [ ] **Step 1: Write the integration test**

`tests/tools/test_tools_integration.py`:
```python
"""Integration test — load tools via three-tier config, execute through SandboxedTool."""

import pytest

from hal.core.event_bus import AsyncEventBus
from hal.tools.protocol import RiskLevel, ToolResult, ToolStatus
from hal.tools.registry import ToolRegistry
from hal.tools.loader import ToolLoader
from hal.tools.sandbox import SandboxedTool


class SimpleTool:
    def __init__(self, name: str, risk: RiskLevel = RiskLevel.LOW) -> None:
        self._name = name
        self._risk = risk

    @property
    def name(self) -> str:
        return self._name

    @property
    def description(self) -> str:
        return f"Tool: {self._name}"

    @property
    def risk_level(self) -> RiskLevel:
        return self._risk

    @property
    def parameters_schema(self) -> dict:
        return {"type": "object"}

    def validate(self, params: dict) -> None:
        pass

    async def execute(self, params: dict) -> ToolResult:
        return ToolResult(status=ToolStatus.SUCCESS, data={"tool": self._name})

    async def rollback(self, params: dict) -> None:
        pass


async def test_full_tool_lifecycle():
    """Register → load via config → wrap in sandbox → execute → verify audit."""
    # Setup
    event_bus = AsyncEventBus()
    registry = ToolRegistry()
    audit_log: list[dict] = []

    async def on_tool_called(data: dict) -> None:
        audit_log.append(data)

    event_bus.subscribe("tool.called", on_tool_called)

    # Register tools
    for name in ["git", "filesystem", "build", "search", "deploy"]:
        risk = RiskLevel.HIGH if name == "deploy" else RiskLevel.LOW
        registry.register_tool(SimpleTool(name, risk))
    registry.register_toolkit("mobile_dev", ["git", "filesystem", "build", "search"])

    # Load via three-tier config (Tier 2: add deploy, remove build)
    loader = ToolLoader(registry)
    tools = loader.load({
        "toolkit": "mobile_dev",
        "tools_add": ["deploy"],
        "tools_remove": ["build"],
    })
    names = sorted(t.name for t in tools)
    assert names == ["deploy", "filesystem", "git", "search"]

    # Wrap in SandboxedTool
    sandboxed = [SandboxedTool(t, event_bus) for t in tools]

    # Execute a low-risk tool
    git_tool = next(t for t in sandboxed if t.name == "git")
    result = await git_tool.execute({"action": "commit"})
    assert result.status == ToolStatus.SUCCESS

    # Execute a high-risk tool — should return PENDING
    deploy_tool = next(t for t in sandboxed if t.name == "deploy")
    result = await deploy_tool.execute({"env": "production"})
    assert result.status == ToolStatus.PENDING
    assert "confirmation_request" in result.data

    # Verify audit trail
    assert len(audit_log) == 2
    assert audit_log[0]["tool_name"] == "git"
    assert audit_log[0]["result_status"] == "success"
    assert audit_log[1]["tool_name"] == "deploy"
    assert audit_log[1]["result_status"] == "pending"
```

- [ ] **Step 2: Run integration test**

Run: `pytest tests/tools/test_tools_integration.py -v`
Expected: 1 passed

- [ ] **Step 3: Run full test suite**

Run: `pytest tests/ -v`
Expected: All tests pass

- [ ] **Step 4: Commit**

```bash
git add tests/tools/test_tools_integration.py
git commit -m "test: add Tool Arsenal integration test"
```

---

## Summary

| Task | Component | Test count (est.) |
|------|-----------|-------------------|
| 1 | Tool Protocol | 5 |
| 2 | Tool Registry | 7 |
| 3 | Tool Loader | 7 |
| 4 | SandboxedTool | 9 |
| 5 | Integration | 1 |
| **Total** | | **~29** |
