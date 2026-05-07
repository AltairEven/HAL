# Plan 3: Identity & Role — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build Layer 3 — role profile parsing, boundary enforcement via system prompt injection, knowledge preloading, and system prompt builder.

**Architecture:** A RoleProfile dataclass encapsulates identity, boundaries, knowledge, concurrency, safeguards, memory, and tools config — all parsed from YAML. The PromptBuilder constructs the system prompt by combining persona, boundaries, context docs, and dynamically injected memories. Tool filtering is driven by the role's tools config via the ToolLoader from Plan 2.

**Tech Stack:** Python 3.9+, PyYAML, pytest

**Spec:** `docs/superpowers/specs/2026-04-27-hal-design.md` — Layer 3: Identity & Role

**Depends on:** Plan 1 (Config), Plan 2 (ToolLoader)

---

## File Structure

```
src/hal/identity/
├── __init__.py
├── role.py              # RoleProfile dataclass + YAML parser
└── prompt_builder.py    # System prompt construction

tests/identity/
├── __init__.py
├── test_role.py
├── test_prompt_builder.py
└── test_identity_integration.py
```

---

## Task 1: Role Profile Model

**Files:**
- Create: `src/hal/identity/__init__.py`
- Create: `src/hal/identity/role.py`
- Create: `tests/identity/__init__.py`
- Create: `tests/identity/test_role.py`

- [ ] **Step 1: Write the failing tests**

`tests/identity/test_role.py`:
```python
from pathlib import Path

import pytest
import yaml

from hal.identity.role import (
    BoundaryConfig,
    ConcurrencyConfig,
    IdentityConfig,
    KnowledgeConfig,
    MemoryConfig,
    RoleProfile,
    SafeguardConfig,
    ToolsConfig,
    load_role,
)


SAMPLE_ROLE_YAML = """
identity:
  name: "Senior iOS Developer"
  persona: |
    You are a senior iOS developer with 8 years of experience.
    You write clean, testable Swift code following SOLID principles.

boundaries:
  can_do:
    - "Write and modify Swift / Objective-C code"
    - "Run Xcode builds and unit tests"
  cannot_do:
    - "Modify Android or HarmonyOS code"
    - "Make product decisions"
  escalate_when:
    - "Requirement is ambiguous or contradictory"
    - "Estimated effort exceeds 2 days"

knowledge:
  domains:
    - ios
    - swift
    - cocoapods
  context_docs:
    - path: "/project/docs/ios-architecture.md"

concurrency:
  max_concurrent_tasks: 1

safeguards:
  max_steps: 50
  cost_budget: 5.0
  max_escalations_per_task: 3
  suspend_ttl: 86400

memory:
  max_episodic: 200
  max_shared: 500

tools:
  toolkit: mobile_dev
  tools_add:
    - cocoapods
  tools_remove:
    - deploy
"""


def test_load_role_from_yaml(tmp_path: Path):
    role_file = tmp_path / "ios_dev.yaml"
    role_file.write_text(SAMPLE_ROLE_YAML)

    role = load_role(role_file)

    assert isinstance(role, RoleProfile)
    assert role.identity.name == "Senior iOS Developer"
    assert "senior iOS developer" in role.identity.persona


def test_role_boundaries(tmp_path: Path):
    role_file = tmp_path / "ios_dev.yaml"
    role_file.write_text(SAMPLE_ROLE_YAML)

    role = load_role(role_file)

    assert len(role.boundaries.can_do) == 2
    assert len(role.boundaries.cannot_do) == 2
    assert len(role.boundaries.escalate_when) == 2
    assert "Swift" in role.boundaries.can_do[0]


def test_role_knowledge(tmp_path: Path):
    role_file = tmp_path / "ios_dev.yaml"
    role_file.write_text(SAMPLE_ROLE_YAML)

    role = load_role(role_file)

    assert role.knowledge.domains == ["ios", "swift", "cocoapods"]
    assert role.knowledge.context_docs[0]["path"] == "/project/docs/ios-architecture.md"


def test_role_concurrency(tmp_path: Path):
    role_file = tmp_path / "ios_dev.yaml"
    role_file.write_text(SAMPLE_ROLE_YAML)

    role = load_role(role_file)
    assert role.concurrency.max_concurrent_tasks == 1


def test_role_safeguards(tmp_path: Path):
    role_file = tmp_path / "ios_dev.yaml"
    role_file.write_text(SAMPLE_ROLE_YAML)

    role = load_role(role_file)

    assert role.safeguards.max_steps == 50
    assert role.safeguards.cost_budget == 5.0
    assert role.safeguards.max_escalations_per_task == 3
    assert role.safeguards.suspend_ttl == 86400


def test_role_memory(tmp_path: Path):
    role_file = tmp_path / "ios_dev.yaml"
    role_file.write_text(SAMPLE_ROLE_YAML)

    role = load_role(role_file)

    assert role.memory.max_episodic == 200
    assert role.memory.max_shared == 500


def test_role_tools_config(tmp_path: Path):
    role_file = tmp_path / "ios_dev.yaml"
    role_file.write_text(SAMPLE_ROLE_YAML)

    role = load_role(role_file)

    assert role.tools.toolkit == "mobile_dev"
    assert role.tools.tools_add == ["cocoapods"]
    assert role.tools.tools_remove == ["deploy"]


def test_role_defaults(tmp_path: Path):
    """Minimal role with only identity — all other sections use defaults."""
    minimal = """
identity:
  name: "Minimal Role"
  persona: "You are a minimal agent."
"""
    role_file = tmp_path / "minimal.yaml"
    role_file.write_text(minimal)

    role = load_role(role_file)

    assert role.identity.name == "Minimal Role"
    assert role.boundaries.can_do == []
    assert role.boundaries.cannot_do == []
    assert role.boundaries.escalate_when == []
    assert role.knowledge.domains == []
    assert role.knowledge.context_docs == []
    assert role.concurrency.max_concurrent_tasks == 1
    assert role.safeguards.max_steps == 50
    assert role.safeguards.cost_budget == 5.0
    assert role.safeguards.max_escalations_per_task == 3
    assert role.safeguards.suspend_ttl == 86400
    assert role.memory.max_episodic == 200
    assert role.memory.max_shared == 500
    assert role.tools.toolkit is None
    assert role.tools.tools_add == []
    assert role.tools.tools_remove == []


def test_role_missing_identity_raises(tmp_path: Path):
    bad_yaml = """
boundaries:
  can_do: ["something"]
"""
    role_file = tmp_path / "bad.yaml"
    role_file.write_text(bad_yaml)

    with pytest.raises(ValueError, match="identity"):
        load_role(role_file)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/identity/test_role.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`src/hal/identity/__init__.py`:
```python
"""HAL Identity & Role — structured role profiles for specialist agents."""
```

`src/hal/identity/role.py`:
```python
"""Role profile — structural constraint system for specialist agents.

A role is not just a system prompt — it drives tool filtering,
boundary enforcement, and knowledge preloading.
"""

from dataclasses import dataclass, field
from pathlib import Path
from typing import Any

import yaml


@dataclass
class IdentityConfig:
    name: str
    persona: str


@dataclass
class BoundaryConfig:
    can_do: list[str] = field(default_factory=list)
    cannot_do: list[str] = field(default_factory=list)
    escalate_when: list[str] = field(default_factory=list)


@dataclass
class KnowledgeConfig:
    domains: list[str] = field(default_factory=list)
    context_docs: list[dict[str, str]] = field(default_factory=list)


@dataclass
class ConcurrencyConfig:
    max_concurrent_tasks: int = 1


@dataclass
class SafeguardConfig:
    max_steps: int = 50
    cost_budget: float = 5.0
    max_escalations_per_task: int = 3
    suspend_ttl: int = 86400


@dataclass
class MemoryConfig:
    max_episodic: int = 200
    max_shared: int = 500


@dataclass
class ToolsConfig:
    toolkit: str | None = None
    tools_add: list[str] = field(default_factory=list)
    tools_remove: list[str] = field(default_factory=list)


@dataclass
class RoleProfile:
    """Complete role profile for a HAL agent instance."""

    identity: IdentityConfig
    boundaries: BoundaryConfig = field(default_factory=BoundaryConfig)
    knowledge: KnowledgeConfig = field(default_factory=KnowledgeConfig)
    concurrency: ConcurrencyConfig = field(default_factory=ConcurrencyConfig)
    safeguards: SafeguardConfig = field(default_factory=SafeguardConfig)
    memory: MemoryConfig = field(default_factory=MemoryConfig)
    tools: ToolsConfig = field(default_factory=ToolsConfig)


def load_role(path: Path) -> RoleProfile:
    """Load a RoleProfile from a YAML file."""
    raw = yaml.safe_load(path.read_text(encoding="utf-8")) or {}

    if "identity" not in raw:
        raise ValueError("Role profile must have an 'identity' section")

    identity_raw = raw["identity"]
    identity = IdentityConfig(
        name=identity_raw["name"],
        persona=identity_raw.get("persona", ""),
    )

    boundaries_raw = raw.get("boundaries", {})
    boundaries = BoundaryConfig(
        can_do=boundaries_raw.get("can_do", []),
        cannot_do=boundaries_raw.get("cannot_do", []),
        escalate_when=boundaries_raw.get("escalate_when", []),
    )

    knowledge_raw = raw.get("knowledge", {})
    knowledge = KnowledgeConfig(
        domains=knowledge_raw.get("domains", []),
        context_docs=knowledge_raw.get("context_docs", []),
    )

    concurrency_raw = raw.get("concurrency", {})
    concurrency = ConcurrencyConfig(
        max_concurrent_tasks=concurrency_raw.get("max_concurrent_tasks", 1),
    )

    safeguards_raw = raw.get("safeguards", {})
    safeguards = SafeguardConfig(
        max_steps=safeguards_raw.get("max_steps", 50),
        cost_budget=safeguards_raw.get("cost_budget", 5.0),
        max_escalations_per_task=safeguards_raw.get("max_escalations_per_task", 3),
        suspend_ttl=safeguards_raw.get("suspend_ttl", 86400),
    )

    memory_raw = raw.get("memory", {})
    memory = MemoryConfig(
        max_episodic=memory_raw.get("max_episodic", 200),
        max_shared=memory_raw.get("max_shared", 500),
    )

    tools_raw = raw.get("tools", {})
    tools = ToolsConfig(
        toolkit=tools_raw.get("toolkit"),
        tools_add=tools_raw.get("tools_add", []),
        tools_remove=tools_raw.get("tools_remove", []),
    )

    return RoleProfile(
        identity=identity,
        boundaries=boundaries,
        knowledge=knowledge,
        concurrency=concurrency,
        safeguards=safeguards,
        memory=memory,
        tools=tools,
    )
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/identity/test_role.py -v`
Expected: 9 passed

- [ ] **Step 5: Commit**

```bash
git add src/hal/identity/ tests/identity/
git commit -m "feat: add RoleProfile model with YAML parsing and defaults"
```

---

## Task 2: System Prompt Builder

**Files:**
- Create: `src/hal/identity/prompt_builder.py`
- Create: `tests/identity/test_prompt_builder.py`

- [ ] **Step 1: Write the failing tests**

`tests/identity/test_prompt_builder.py`:
```python
import pytest

from hal.identity.prompt_builder import PromptBuilder
from hal.identity.role import (
    BoundaryConfig,
    IdentityConfig,
    KnowledgeConfig,
    RoleProfile,
)


def _make_role(
    name: str = "Senior iOS Developer",
    persona: str = "You are a senior iOS developer.",
    can_do: list[str] | None = None,
    cannot_do: list[str] | None = None,
    escalate_when: list[str] | None = None,
) -> RoleProfile:
    return RoleProfile(
        identity=IdentityConfig(name=name, persona=persona),
        boundaries=BoundaryConfig(
            can_do=can_do or ["Write Swift code"],
            cannot_do=cannot_do or ["Modify Android code"],
            escalate_when=escalate_when or ["Requirement is ambiguous"],
        ),
    )


def test_prompt_contains_persona():
    role = _make_role(persona="You are a senior iOS developer with deep expertise.")
    builder = PromptBuilder(role)
    prompt = builder.build()

    assert "senior iOS developer with deep expertise" in prompt


def test_prompt_contains_boundaries():
    role = _make_role(
        can_do=["Write Swift code", "Run tests"],
        cannot_do=["Modify Android code"],
        escalate_when=["Change impacts public API"],
    )
    builder = PromptBuilder(role)
    prompt = builder.build()

    assert "Write Swift code" in prompt
    assert "Run tests" in prompt
    assert "Modify Android code" in prompt
    assert "Change impacts public API" in prompt


def test_prompt_contains_role_name():
    role = _make_role(name="QA Engineer")
    builder = PromptBuilder(role)
    prompt = builder.build()

    assert "QA Engineer" in prompt


def test_prompt_with_context_docs(tmp_path):
    doc_file = tmp_path / "arch.md"
    doc_file.write_text("# Architecture\nUse MVVM pattern.")

    role = RoleProfile(
        identity=IdentityConfig(name="Dev", persona="You are a developer."),
        knowledge=KnowledgeConfig(
            domains=["ios"],
            context_docs=[{"path": str(doc_file)}],
        ),
    )
    builder = PromptBuilder(role)
    prompt = builder.build()

    assert "MVVM pattern" in prompt


def test_prompt_with_context_doc_missing_file():
    role = RoleProfile(
        identity=IdentityConfig(name="Dev", persona="You are a developer."),
        knowledge=KnowledgeConfig(
            context_docs=[{"path": "/nonexistent/doc.md"}],
        ),
    )
    builder = PromptBuilder(role)
    prompt = builder.build()

    # Should not crash — missing docs are skipped with a warning in prompt
    assert "Dev" in prompt


def test_prompt_with_memories_injection():
    role = _make_role()
    builder = PromptBuilder(role)
    memories = [
        "CocoaPods 1.15 is incompatible with Xcode 16.",
        "Always run tests before committing.",
    ]
    prompt = builder.build(memories=memories)

    assert "CocoaPods 1.15" in prompt
    assert "Always run tests" in prompt


def test_prompt_without_memories():
    role = _make_role()
    builder = PromptBuilder(role)
    prompt = builder.build()

    # No memories section when empty
    assert "Relevant memories" not in prompt


def test_prompt_section_ordering():
    """Persona comes before boundaries, boundaries before knowledge."""
    role = RoleProfile(
        identity=IdentityConfig(name="Dev", persona="PERSONA_MARKER"),
        boundaries=BoundaryConfig(can_do=["CANDO_MARKER"]),
        knowledge=KnowledgeConfig(domains=["DOMAIN_MARKER"]),
    )
    builder = PromptBuilder(role)
    prompt = builder.build()

    persona_pos = prompt.index("PERSONA_MARKER")
    cando_pos = prompt.index("CANDO_MARKER")
    domain_pos = prompt.index("DOMAIN_MARKER")

    assert persona_pos < cando_pos < domain_pos
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/identity/test_prompt_builder.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`src/hal/identity/prompt_builder.py`:
```python
"""System prompt builder — constructs the full system prompt from a RoleProfile.

Combines: persona + boundaries + context_docs (static base)
        + memories (dynamic tail, injected per task).
"""

import logging
from pathlib import Path

from hal.identity.role import RoleProfile

logger = logging.getLogger(__name__)


class PromptBuilder:
    """Builds a system prompt from a RoleProfile and optional dynamic memories."""

    def __init__(self, role: RoleProfile) -> None:
        self._role = role

    def build(self, *, memories: list[str] | None = None) -> str:
        """Build the complete system prompt.

        Args:
            memories: Optional list of memory summaries to inject (dynamic tail).
        """
        sections: list[str] = []

        # Identity
        sections.append(f"# Role: {self._role.identity.name}\n")
        sections.append(self._role.identity.persona.strip())

        # Boundaries
        b = self._role.boundaries
        if b.can_do or b.cannot_do or b.escalate_when:
            sections.append("\n## Boundaries\n")
            if b.can_do:
                sections.append("**You CAN:**")
                for item in b.can_do:
                    sections.append(f"- {item}")
            if b.cannot_do:
                sections.append("\n**You CANNOT:**")
                for item in b.cannot_do:
                    sections.append(f"- {item}")
            if b.escalate_when:
                sections.append("\n**Escalate when:**")
                for item in b.escalate_when:
                    sections.append(f"- {item}")

        # Knowledge — domains
        k = self._role.knowledge
        if k.domains:
            sections.append(f"\n## Expertise Domains\n")
            sections.append(", ".join(k.domains))

        # Knowledge — context docs
        if k.context_docs:
            sections.append("\n## Reference Documentation\n")
            for doc in k.context_docs:
                doc_path = Path(doc["path"])
                if doc_path.exists():
                    content = doc_path.read_text(encoding="utf-8").strip()
                    sections.append(f"### {doc_path.name}\n{content}")
                else:
                    sections.append(
                        f"[Warning: document not found: {doc['path']}]"
                    )
                    logger.warning("Context doc not found: %s", doc["path"])

        # Dynamic memories
        if memories:
            sections.append("\n## Relevant Memories\n")
            for mem in memories:
                sections.append(f"- {mem}")

        return "\n".join(sections)
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/identity/test_prompt_builder.py -v`
Expected: 8 passed

- [ ] **Step 5: Commit**

```bash
git add src/hal/identity/prompt_builder.py tests/identity/test_prompt_builder.py
git commit -m "feat: add PromptBuilder for system prompt construction"
```

---

## Task 3: Integration Test

**Files:**
- Create: `tests/identity/test_identity_integration.py`

- [ ] **Step 1: Write the integration test**

`tests/identity/test_identity_integration.py`:
```python
"""Integration: parse role YAML → build prompt → load tools via ToolLoader."""

from pathlib import Path

import pytest
import yaml

from hal.core.event_bus import AsyncEventBus
from hal.identity.prompt_builder import PromptBuilder
from hal.identity.role import load_role
from hal.tools.loader import ToolLoader
from hal.tools.protocol import RiskLevel, ToolResult, ToolStatus
from hal.tools.registry import ToolRegistry
from hal.tools.sandbox import SandboxedTool


class StubTool:
    def __init__(self, name: str, risk: RiskLevel = RiskLevel.LOW) -> None:
        self._name = name
        self._risk = risk

    @property
    def name(self) -> str:
        return self._name

    @property
    def description(self) -> str:
        return f"Stub: {self._name}"

    @property
    def risk_level(self) -> RiskLevel:
        return self._risk

    @property
    def parameters_schema(self) -> dict:
        return {}

    def validate(self, params: dict) -> None:
        pass

    async def execute(self, params: dict) -> ToolResult:
        return ToolResult(status=ToolStatus.SUCCESS, data={"tool": self._name})

    async def rollback(self, params: dict) -> None:
        pass


async def test_role_to_prompt_to_tools(tmp_path: Path):
    """Full lifecycle: YAML → RoleProfile → PromptBuilder → ToolLoader."""
    # Write role file
    role_yaml = tmp_path / "ios_dev.yaml"
    role_yaml.write_text(yaml.dump({
        "identity": {
            "name": "Senior iOS Developer",
            "persona": "You are a senior iOS developer.",
        },
        "boundaries": {
            "can_do": ["Write Swift code"],
            "cannot_do": ["Modify Android code"],
            "escalate_when": ["Requirement is ambiguous"],
        },
        "knowledge": {"domains": ["ios", "swift"]},
        "tools": {
            "toolkit": "mobile_dev",
            "tools_add": ["cocoapods"],
            "tools_remove": ["deploy"],
        },
    }))

    # Parse role
    role = load_role(role_yaml)
    assert role.identity.name == "Senior iOS Developer"

    # Build prompt
    builder = PromptBuilder(role)
    prompt = builder.build(memories=["Use CocoaPods 1.14, not 1.15."])
    assert "Senior iOS Developer" in prompt
    assert "Write Swift code" in prompt
    assert "Modify Android code" in prompt
    assert "CocoaPods 1.14" in prompt

    # Load tools
    event_bus = AsyncEventBus()
    registry = ToolRegistry()
    for name in ["git", "filesystem", "build", "search", "deploy", "cocoapods"]:
        risk = RiskLevel.HIGH if name == "deploy" else RiskLevel.LOW
        registry.register_tool(StubTool(name, risk))
    registry.register_toolkit("mobile_dev", ["git", "filesystem", "build", "search"])

    loader = ToolLoader(registry)
    tools_config = {
        "toolkit": role.tools.toolkit,
        "tools_add": role.tools.tools_add,
        "tools_remove": role.tools.tools_remove,
    }
    tools = loader.load(tools_config)
    sandboxed = [SandboxedTool(t, event_bus) for t in tools]

    tool_names = sorted(t.name for t in sandboxed)
    assert tool_names == ["build", "cocoapods", "filesystem", "git", "search"]
    assert "deploy" not in tool_names  # removed
```

- [ ] **Step 2: Run integration test**

Run: `pytest tests/identity/test_identity_integration.py -v`
Expected: 1 passed

- [ ] **Step 3: Commit**

```bash
git add tests/identity/test_identity_integration.py
git commit -m "test: add Identity & Role integration test"
```

---

## Summary

| Task | Component | Test count (est.) |
|------|-----------|-------------------|
| 1 | Role Profile Model | 9 |
| 2 | System Prompt Builder | 8 |
| 3 | Integration | 1 |
| **Total** | | **~18** |
