# Plan 7: ClaudeCodeDriver — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the V1 production AgentDriver wrapping Claude Code's SDK/headless mode, resolving integration questions around tool coexistence, session history, cost accounting, and cancellation.

**Architecture:** ClaudeCodeDriver implements the AgentDriver protocol, delegating LLM execution to Claude Code. It translates HAL's context (Working Memory, SandboxedTools, system prompt, constraints) into Claude Code SDK calls, and maps results back into DriverResult. This plan starts with an SDK discovery spike to resolve open questions before implementation.

**Tech Stack:** Python 3.9+, Claude Agent SDK, pytest, pytest-asyncio

**Spec:** `docs/superpowers/specs/2026-04-27-hal-design.md` — ClaudeCodeDriver section in Layer 1

**Depends on:** Plan 4 (AgentDriver protocol, TaskRunner)

---

## File Structure

```
src/hal/drivers/
├── __init__.py          # (exists from Plan 4)
└── claude_code.py       # ClaudeCodeDriver implementation

tests/drivers/
├── __init__.py          # (exists from Plan 4)
└── test_claude_code.py  # ClaudeCodeDriver tests

docs/
└── claude-code-sdk-findings.md   # SDK discovery spike findings
```

---

## Important Note

This plan differs from Plans 1-6. The ClaudeCodeDriver depends on the Claude Agent SDK's actual API surface, which has open questions documented in the spec:

1. **Tool coexistence** — Can built-in tools be disabled/overridden?
2. **Session history interop** — Is conversation history serializable?
3. **Cost/steps accounting** — How opaque is token consumption?
4. **Cancellation support** — Is mid-execution interruption possible?

Task 1 (SDK Discovery) resolves these questions. Subsequent tasks adapt based on findings. Code in Tasks 2-7 uses placeholder patterns that MUST be updated based on Task 1 findings.

---

## Task 1: SDK Discovery & Integration Spike

**Files:**
- Create: `docs/claude-code-sdk-findings.md`

- [ ] **Step 1: Install Claude Agent SDK**

Run: `pip install claude-code-sdk` (or the appropriate package name)

- [ ] **Step 2: Explore SDK API surface**

Write a discovery script to answer the four open questions:

```python
"""SDK discovery spike — do NOT commit this file."""

# Question 1: Tool configuration
# Can we pass custom tools? Can built-in tools be disabled?
# Try: create a session with custom tool definitions

# Question 2: Session history
# Can we extract/inject conversation history?
# Try: run a session, inspect returned messages

# Question 3: Cost reporting
# Does the SDK report token usage?
# Try: inspect session result for usage stats

# Question 4: Cancellation
# Is there a way to cancel mid-execution?
# Try: check for cancel/abort methods or async cancellation
```

- [ ] **Step 3: Document findings**

Create `docs/claude-code-sdk-findings.md` with:
- SDK version tested
- API surface summary
- Answer to each of the 4 questions
- Chosen integration strategy for each
- Any SDK limitations or workarounds needed

- [ ] **Step 4: Commit findings**

```bash
git add docs/claude-code-sdk-findings.md
git commit -m "docs: Claude Code SDK integration spike findings"
```

---

## Task 2: ClaudeCodeDriver — Basic Execution

**Files:**
- Create: `src/hal/drivers/claude_code.py`
- Create: `tests/drivers/test_claude_code.py`

- [ ] **Step 1: Write the failing test**

`tests/drivers/test_claude_code.py`:
```python
import pytest

from hal.core.driver import DriverResult, DriverStatus
from hal.drivers.claude_code import ClaudeCodeDriver


@pytest.mark.skipif(
    not _sdk_available(),
    reason="Claude Code SDK not installed or API key not configured",
)
async def test_basic_execution():
    driver = ClaudeCodeDriver()

    result = await driver.run(
        messages=[{"role": "user", "content": "Reply with exactly: HELLO"}],
        tools=[],
        system_prompt="You are a test agent. Follow instructions exactly.",
        max_steps=5,
        cost_budget=0.1,
    )

    assert result.status == DriverStatus.COMPLETED
    assert result.steps_taken >= 1
    assert result.token_usage > 0
    assert len(result.messages) > 0


def _sdk_available() -> bool:
    try:
        import claude_code_sdk  # adjust based on actual package name
        return True
    except ImportError:
        return False
```

- [ ] **Step 2: Write minimal implementation**

`src/hal/drivers/claude_code.py`:
```python
"""ClaudeCodeDriver — V1 production Driver wrapping Claude Code SDK.

Translates HAL's AgentDriver protocol into Claude Code SDK calls.
"""

import logging
from typing import Any

from hal.core.driver import (
    CancellationToken,
    DriverResult,
    DriverStatus,
    Lesson,
    SuspendContext,
)

logger = logging.getLogger(__name__)

# NOTE: Import path and API usage MUST be updated based on Task 1 findings.
# The code below uses placeholder patterns.


class ClaudeCodeDriver:
    """AgentDriver wrapping Claude Code SDK/headless mode.

    Integration strategy (to be finalized after SDK discovery):
    - Tool coexistence: [UPDATE BASED ON TASK 1]
    - Session history: [UPDATE BASED ON TASK 1]
    - Cost accounting: best-effort from SDK reports
    - Cancellation: [UPDATE BASED ON TASK 1]
    """

    def __init__(self, **sdk_options: Any) -> None:
        self._sdk_options = sdk_options

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
        """Execute via Claude Code SDK.

        Translates:
        - messages → SDK conversation input
        - tools → SDK tool definitions (JSON Schema)
        - system_prompt → SDK system prompt
        - max_steps/cost_budget → SDK constraints (if supported)
        """
        try:
            # TODO: Replace with actual SDK calls based on Task 1 findings
            # Placeholder structure:
            #
            # session = await sdk.create_session(
            #     system_prompt=system_prompt,
            #     tools=self._translate_tools(tools),
            #     messages=messages,
            #     max_turns=max_steps,
            # )
            #
            # result = await session.run()
            #
            # return DriverResult(
            #     status=self._map_status(result.status),
            #     messages=result.messages,
            #     steps_taken=result.turns_used,
            #     token_usage=result.total_tokens,
            #     summary=result.final_response,
            # )

            raise NotImplementedError(
                "ClaudeCodeDriver requires SDK discovery (Task 1) to be completed first. "
                "Update this implementation based on docs/claude-code-sdk-findings.md"
            )

        except NotImplementedError:
            raise
        except Exception as e:
            logger.exception("ClaudeCodeDriver execution failed")
            return DriverResult(
                status=DriverStatus.FAILED,
                messages=messages,
                steps_taken=0,
                token_usage=0,
                summary=f"Driver error: {e}",
            )

    def _translate_tools(self, tools: list[Any]) -> list[dict[str, Any]]:
        """Translate SandboxedTool wrappers to SDK tool format."""
        sdk_tools = []
        for tool in tools:
            sdk_tools.append({
                "name": tool.name,
                "description": tool.description,
                "parameters": tool.parameters_schema,
            })
        return sdk_tools

    @staticmethod
    def _map_status(sdk_status: str) -> DriverStatus:
        """Map SDK status to HAL DriverStatus."""
        mapping = {
            "completed": DriverStatus.COMPLETED,
            "error": DriverStatus.FAILED,
            "stopped": DriverStatus.SUSPENDED,
        }
        return mapping.get(sdk_status, DriverStatus.FAILED)
```

- [ ] **Step 3: Run test**

Run: `pytest tests/drivers/test_claude_code.py -v`
Expected: Test skipped (SDK not available) or passes if SDK is configured.

- [ ] **Step 4: Commit**

```bash
git add src/hal/drivers/claude_code.py tests/drivers/test_claude_code.py
git commit -m "feat: add ClaudeCodeDriver scaffold (pending SDK discovery)"
```

---

## Task 3: Tool Coexistence

**Files:**
- Modify: `src/hal/drivers/claude_code.py`
- Modify: `tests/drivers/test_claude_code.py`

- [ ] **Step 1: Implement tool translation based on SDK findings**

Update `_translate_tools()` and the `run()` method based on the chosen strategy from Task 1:

**Strategy A (preferred): Selective disable** — Disable Claude Code's built-in tools, inject only SandboxedTool wrappers.

**Strategy B: Proxy** — Wrap Claude Code's built-in tools through SandboxedTool for audit/visibility.

**Strategy C: Coexistence** — Accept that built-in tools bypass HAL's tool layer; rely on role boundaries for logical isolation.

- [ ] **Step 2: Add test for tool execution through SandboxedTool**

```python
# Add to tests/drivers/test_claude_code.py

@pytest.mark.skipif(not _sdk_available(), reason="SDK not available")
async def test_tool_execution_through_sandbox():
    """Verify tools execute through SandboxedTool wrappers, not raw SDK tools."""
    # Test that HAL-provided tools are called via the SandboxedTool interface
    # (audit events emitted, risk checks applied)
    pass  # UPDATE based on SDK findings
```

- [ ] **Step 3: Commit**

```bash
git add src/hal/drivers/claude_code.py tests/drivers/test_claude_code.py
git commit -m "feat: implement tool coexistence strategy for ClaudeCodeDriver"
```

---

## Task 4: Session History Interop

**Files:**
- Modify: `src/hal/drivers/claude_code.py`
- Modify: `tests/drivers/test_claude_code.py`

- [ ] **Step 1: Implement history round-trip**

Update `run()` to:
1. Inject Working Memory messages into SDK session
2. Extract updated messages from SDK result
3. Return in DriverResult.messages for Working Memory update

- [ ] **Step 2: Add test for checkpoint/resume with history**

```python
@pytest.mark.skipif(not _sdk_available(), reason="SDK not available")
async def test_history_roundtrip():
    """Messages from first run can be passed back for resume."""
    driver = ClaudeCodeDriver()

    r1 = await driver.run(
        messages=[{"role": "user", "content": "Step 1"}],
        tools=[], system_prompt="Test", max_steps=3, cost_budget=0.1,
    )

    # Use r1.messages as input for second run (simulating resume)
    r2 = await driver.run(
        messages=r1.messages + [{"role": "user", "content": "Step 2"}],
        tools=[], system_prompt="Test", max_steps=3, cost_budget=0.1,
    )

    assert r2.status == DriverStatus.COMPLETED
    assert len(r2.messages) > len(r1.messages)
```

- [ ] **Step 3: Commit**

```bash
git add src/hal/drivers/claude_code.py tests/drivers/test_claude_code.py
git commit -m "feat: implement session history interop for ClaudeCodeDriver"
```

---

## Task 5: Cost & Steps Accounting

**Files:**
- Modify: `src/hal/drivers/claude_code.py`

- [ ] **Step 1: Implement cost extraction**

Extract token usage from SDK result. Map to DriverResult fields:
- `steps_taken`: number of LLM turns/tool calls
- `token_usage`: total tokens consumed (input + output)

Best-effort — trust the SDK's self-reported numbers.

- [ ] **Step 2: Add test**

```python
@pytest.mark.skipif(not _sdk_available(), reason="SDK not available")
async def test_cost_reporting():
    driver = ClaudeCodeDriver()
    result = await driver.run(
        messages=[{"role": "user", "content": "Say hello"}],
        tools=[], system_prompt="Test", max_steps=5, cost_budget=0.1,
    )
    assert result.token_usage > 0
    assert result.steps_taken >= 1
```

- [ ] **Step 3: Commit**

```bash
git add src/hal/drivers/claude_code.py tests/drivers/test_claude_code.py
git commit -m "feat: add cost/steps accounting to ClaudeCodeDriver"
```

---

## Task 6: Cancellation Support

**Files:**
- Modify: `src/hal/drivers/claude_code.py`

- [ ] **Step 1: Implement cancellation based on SDK findings**

**If SDK supports cancellation:** Check `cancellation_token.is_cancelled` between SDK steps. Return `DriverStatus.SUSPENDED` when cancelled.

**If SDK does NOT support cancellation (soft preemption):** Document limitation. Return normally — preemption takes effect after Driver completes.

- [ ] **Step 2: Add test**

```python
@pytest.mark.skipif(not _sdk_available(), reason="SDK not available")
async def test_cancellation():
    driver = ClaudeCodeDriver()
    token = CancellationToken()
    token.cancel()

    result = await driver.run(
        messages=[{"role": "user", "content": "Long task"}],
        tools=[], system_prompt="Test", max_steps=50, cost_budget=1.0,
        cancellation_token=token,
    )

    # If SDK supports cancellation: SUSPENDED
    # If not: COMPLETED (soft preemption)
    assert result.status in (DriverStatus.SUSPENDED, DriverStatus.COMPLETED)
```

- [ ] **Step 3: Commit**

```bash
git add src/hal/drivers/claude_code.py tests/drivers/test_claude_code.py
git commit -m "feat: add cancellation support to ClaudeCodeDriver"
```

---

## Task 7: Lesson Extraction

**Files:**
- Modify: `src/hal/drivers/claude_code.py`

- [ ] **Step 1: Implement lesson extraction**

After SDK execution, scan the result for extractable lessons:
- Tool failures → generate lessons with trigger="tool_failure"
- Retries/re-planning → generate lessons about what didn't work
- Successful patterns → generate positive lessons

Populate `DriverResult.lessons` field.

- [ ] **Step 2: Add test**

```python
@pytest.mark.skipif(not _sdk_available(), reason="SDK not available")
async def test_lesson_extraction():
    driver = ClaudeCodeDriver()
    result = await driver.run(
        messages=[{"role": "user", "content": "Install numpy and verify"}],
        tools=[], system_prompt="Test", max_steps=10, cost_budget=0.5,
    )
    # Lessons may or may not be extracted depending on execution
    assert isinstance(result.lessons, list)
```

- [ ] **Step 3: Commit**

```bash
git add src/hal/drivers/claude_code.py tests/drivers/test_claude_code.py
git commit -m "feat: add lesson extraction to ClaudeCodeDriver"
```

---

## Task 8: Integration Test

**Files:**
- Modify: `tests/drivers/test_claude_code.py`

- [ ] **Step 1: Write full lifecycle integration test**

```python
@pytest.mark.skipif(not _sdk_available(), reason="SDK not available")
async def test_full_lifecycle_with_task_runner():
    """Full TaskRunner lifecycle with ClaudeCodeDriver."""
    from hal.core.event_bus import AsyncEventBus
    from hal.identity.role import IdentityConfig, RoleProfile, SafeguardConfig
    from hal.runner.checkpoint import InMemoryCheckpointStore
    from hal.runner.task_runner import TaskRunner

    driver = ClaudeCodeDriver()
    role = RoleProfile(
        identity=IdentityConfig(name="Test", persona="Test agent."),
        safeguards=SafeguardConfig(max_steps=10, cost_budget=1.0),
    )

    runner = TaskRunner(
        agent_id="test-agent",
        driver=driver,
        role=role,
        event_bus=AsyncEventBus(),
        checkpoint_store=InMemoryCheckpointStore(),
        hard_cost_ceiling=50.0,
    )

    result = await runner.run(
        task_id="integration-test",
        task_description="Reply with exactly: INTEGRATION_TEST_PASS",
        trace_id="tr-int",
    )

    assert result.status == DriverStatus.COMPLETED
    assert result.steps_taken > 0
```

- [ ] **Step 2: Run integration test**

Run: `pytest tests/drivers/test_claude_code.py -v`
Expected: Tests pass (or skip if SDK not configured)

- [ ] **Step 3: Commit**

```bash
git add tests/drivers/test_claude_code.py
git commit -m "test: add ClaudeCodeDriver integration test with TaskRunner"
```

---

## Summary

| Task | Component | Test count (est.) |
|------|-----------|-------------------|
| 1 | SDK Discovery | 0 (document) |
| 2 | Basic Execution | 1 |
| 3 | Tool Coexistence | 1 |
| 4 | Session History | 1 |
| 5 | Cost & Steps | 1 |
| 6 | Cancellation | 1 |
| 7 | Lesson Extraction | 1 |
| 8 | Integration | 1 |
| **Total** | | **~7** |

Note: Test counts are lower because most tests require a live SDK connection and are marked `skipif`. The real validation happens during SDK discovery (Task 1) and manual integration testing.
