# Plan 5: Memory System — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the three-tier memory system — Episodic Memory (per-instance), Shared Memory (cluster-wide), tag-based retrieval, and memory hygiene (confidence decay, eviction, consolidation).

**Architecture:** Episodic and Shared Memory share a common MemoryEntry model and storage interface (InMemory + SQLite backends). Retrieval is tag-based (V1). Hygiene runs lazily at retrieval time (confidence decay) and on write (consolidation, cap eviction). Driver-reported lessons are routed by scope (episodic/shared) with confidence caps.

**Tech Stack:** Python 3.9+, sqlite3 (stdlib), pytest

**Spec:** `docs/superpowers/specs/2026-04-27-hal-design.md` — Memory System

**Depends on:** Plan 1 (EventBus, Config), Plan 4 (TaskRunner integration for lesson routing)

---

## File Structure

```
src/hal/memory/
├── __init__.py
├── models.py            # MemoryEntry dataclass
├── store.py             # MemoryStore protocol + InMemory/SQLite backends
├── retrieval.py         # Tag-based retrieval with scoring
├── hygiene.py           # Decay, consolidation, eviction
├── episodic.py          # Per-instance Episodic Memory manager
└── shared.py            # Cluster-wide Shared Memory manager

tests/memory/
├── __init__.py
├── test_models.py
├── test_store.py
├── test_retrieval.py
├── test_hygiene.py
├── test_episodic.py
├── test_shared.py
└── test_memory_integration.py
```

---

## Task 1: Memory Entry Model

**Files:**
- Create: `src/hal/memory/__init__.py`
- Create: `src/hal/memory/models.py`
- Create: `tests/memory/__init__.py`
- Create: `tests/memory/test_models.py`

- [ ] **Step 1: Write the failing tests**

`tests/memory/test_models.py`:
```python
from hal.memory.models import MemoryEntry


def test_create_memory_entry():
    entry = MemoryEntry(
        entry_id="m-1",
        content="CocoaPods 1.15 is incompatible with Xcode 16",
        summary="CocoaPods 1.15 / Xcode 16 incompatibility",
        tags=["cocoapods", "xcode", "build"],
        confidence=0.7,
        source_task="feat-login-sdk",
        trigger="build_failure",
    )
    assert entry.entry_id == "m-1"
    assert entry.confidence == 0.7
    assert entry.times_validated == 0
    assert entry.last_used is None


def test_memory_entry_serialization():
    entry = MemoryEntry(
        entry_id="m-2",
        content="Always pin dependencies",
        summary="Pin deps",
        tags=["deps"],
        confidence=0.5,
        source_task="task-1",
        trigger="dependency_error",
        times_validated=2,
        last_used="2026-04-28T10:00:00Z",
        created_at="2026-04-27T08:00:00Z",
    )
    data = entry.to_dict()
    restored = MemoryEntry.from_dict(data)

    assert restored.entry_id == "m-2"
    assert restored.confidence == 0.5
    assert restored.times_validated == 2
    assert restored.tags == ["deps"]
    assert restored.last_used == "2026-04-28T10:00:00Z"


def test_memory_entry_defaults():
    entry = MemoryEntry(
        entry_id="m-3",
        content="Test",
        summary="Test",
        tags=[],
        confidence=0.7,
        source_task="t",
        trigger="manual",
    )
    assert entry.times_validated == 0
    assert entry.last_used is None
    assert entry.created_at is not None  # auto-set
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/memory/test_models.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`src/hal/memory/__init__.py`:
```python
"""HAL Memory System — episodic and shared memory with tag-based retrieval."""
```

`src/hal/memory/models.py`:
```python
"""Memory entry model — single unit of stored knowledge."""

from dataclasses import dataclass, field
from datetime import datetime, timezone
from typing import Any


@dataclass
class MemoryEntry:
    """A single memory entry (episodic or shared)."""

    entry_id: str
    content: str
    summary: str
    tags: list[str]
    confidence: float
    source_task: str
    trigger: str
    times_validated: int = 0
    last_used: str | None = None
    created_at: str = field(
        default_factory=lambda: datetime.now(timezone.utc).isoformat()
    )
    created_by: str | None = None  # for shared memory — writer instance ID

    def to_dict(self) -> dict[str, Any]:
        return {
            "entry_id": self.entry_id,
            "content": self.content,
            "summary": self.summary,
            "tags": self.tags,
            "confidence": self.confidence,
            "source_task": self.source_task,
            "trigger": self.trigger,
            "times_validated": self.times_validated,
            "last_used": self.last_used,
            "created_at": self.created_at,
            "created_by": self.created_by,
        }

    @classmethod
    def from_dict(cls, data: dict[str, Any]) -> "MemoryEntry":
        return cls(
            entry_id=data["entry_id"],
            content=data["content"],
            summary=data["summary"],
            tags=data["tags"],
            confidence=data["confidence"],
            source_task=data["source_task"],
            trigger=data["trigger"],
            times_validated=data.get("times_validated", 0),
            last_used=data.get("last_used"),
            created_at=data.get("created_at", datetime.now(timezone.utc).isoformat()),
            created_by=data.get("created_by"),
        )
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/memory/test_models.py -v`
Expected: 3 passed

- [ ] **Step 5: Commit**

```bash
git add src/hal/memory/ tests/memory/
git commit -m "feat: add MemoryEntry model with serialization"
```

---

## Task 2: Memory Store (InMemory + SQLite)

**Files:**
- Create: `src/hal/memory/store.py`
- Create: `tests/memory/test_store.py`

- [ ] **Step 1: Write the failing tests**

`tests/memory/test_store.py`:
```python
import pytest

from hal.memory.models import MemoryEntry
from hal.memory.store import InMemoryMemoryStore, SQLiteMemoryStore


def _make_entry(entry_id: str = "m-1", tags: list[str] | None = None, confidence: float = 0.7) -> MemoryEntry:
    return MemoryEntry(
        entry_id=entry_id,
        content="Test content",
        summary="Test",
        tags=tags or ["test"],
        confidence=confidence,
        source_task="task-1",
        trigger="manual",
    )


class TestInMemoryMemoryStore:
    @pytest.fixture
    def store(self) -> InMemoryMemoryStore:
        return InMemoryMemoryStore()

    def test_save_and_get(self, store):
        entry = _make_entry()
        store.save(entry)
        assert store.get("m-1") is not None
        assert store.get("m-1").content == "Test content"

    def test_get_nonexistent(self, store):
        assert store.get("nonexistent") is None

    def test_delete(self, store):
        store.save(_make_entry())
        store.delete("m-1")
        assert store.get("m-1") is None

    def test_list_all(self, store):
        store.save(_make_entry("m-1"))
        store.save(_make_entry("m-2"))
        assert len(store.list_all()) == 2

    def test_find_by_tags(self, store):
        store.save(_make_entry("m-1", tags=["ios", "build"]))
        store.save(_make_entry("m-2", tags=["android", "build"]))
        store.save(_make_entry("m-3", tags=["ios", "test"]))

        results = store.find_by_tags(["ios"])
        assert len(results) == 2
        ids = {r.entry_id for r in results}
        assert ids == {"m-1", "m-3"}

    def test_count(self, store):
        assert store.count() == 0
        store.save(_make_entry("m-1"))
        store.save(_make_entry("m-2"))
        assert store.count() == 2

    def test_overwrite(self, store):
        store.save(_make_entry("m-1", confidence=0.5))
        store.save(_make_entry("m-1", confidence=0.8))
        assert store.get("m-1").confidence == 0.8


class TestSQLiteMemoryStore:
    @pytest.fixture
    def store(self, tmp_path) -> SQLiteMemoryStore:
        return SQLiteMemoryStore(tmp_path / "memory.db")

    def test_save_and_get(self, store):
        entry = _make_entry()
        store.save(entry)
        loaded = store.get("m-1")
        assert loaded is not None
        assert loaded.content == "Test content"
        assert loaded.tags == ["test"]

    def test_get_nonexistent(self, store):
        assert store.get("nonexistent") is None

    def test_delete(self, store):
        store.save(_make_entry())
        store.delete("m-1")
        assert store.get("m-1") is None

    def test_find_by_tags(self, store):
        store.save(_make_entry("m-1", tags=["ios", "build"]))
        store.save(_make_entry("m-2", tags=["android", "build"]))

        results = store.find_by_tags(["ios"])
        assert len(results) == 1
        assert results[0].entry_id == "m-1"

    def test_count(self, store):
        assert store.count() == 0
        store.save(_make_entry())
        assert store.count() == 1
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/memory/test_store.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`src/hal/memory/store.py`:
```python
"""Memory store — pluggable backends for memory persistence."""

import json
import sqlite3
from pathlib import Path
from typing import Protocol

from hal.memory.models import MemoryEntry


class MemoryStore(Protocol):
    def save(self, entry: MemoryEntry) -> None: ...
    def get(self, entry_id: str) -> MemoryEntry | None: ...
    def delete(self, entry_id: str) -> None: ...
    def list_all(self) -> list[MemoryEntry]: ...
    def find_by_tags(self, tags: list[str]) -> list[MemoryEntry]: ...
    def count(self) -> int: ...


class InMemoryMemoryStore:
    """In-memory store for testing."""

    def __init__(self) -> None:
        self._store: dict[str, MemoryEntry] = {}

    def save(self, entry: MemoryEntry) -> None:
        self._store[entry.entry_id] = entry

    def get(self, entry_id: str) -> MemoryEntry | None:
        return self._store.get(entry_id)

    def delete(self, entry_id: str) -> None:
        self._store.pop(entry_id, None)

    def list_all(self) -> list[MemoryEntry]:
        return list(self._store.values())

    def find_by_tags(self, tags: list[str]) -> list[MemoryEntry]:
        tag_set = set(tags)
        return [e for e in self._store.values() if tag_set & set(e.tags)]

    def count(self) -> int:
        return len(self._store)


class SQLiteMemoryStore:
    """SQLite-backed memory store for production."""

    def __init__(self, db_path: Path) -> None:
        self._conn = sqlite3.connect(str(db_path))
        self._conn.execute("PRAGMA journal_mode=WAL")
        self._conn.execute("""
            CREATE TABLE IF NOT EXISTS memories (
                entry_id TEXT PRIMARY KEY,
                data TEXT NOT NULL,
                tags TEXT NOT NULL
            )
        """)
        self._conn.commit()

    def save(self, entry: MemoryEntry) -> None:
        self._conn.execute(
            "INSERT OR REPLACE INTO memories (entry_id, data, tags) VALUES (?, ?, ?)",
            (entry.entry_id, json.dumps(entry.to_dict()), json.dumps(entry.tags)),
        )
        self._conn.commit()

    def get(self, entry_id: str) -> MemoryEntry | None:
        row = self._conn.execute(
            "SELECT data FROM memories WHERE entry_id = ?", (entry_id,)
        ).fetchone()
        if row is None:
            return None
        return MemoryEntry.from_dict(json.loads(row[0]))

    def delete(self, entry_id: str) -> None:
        self._conn.execute("DELETE FROM memories WHERE entry_id = ?", (entry_id,))
        self._conn.commit()

    def list_all(self) -> list[MemoryEntry]:
        rows = self._conn.execute("SELECT data FROM memories").fetchall()
        return [MemoryEntry.from_dict(json.loads(r[0])) for r in rows]

    def find_by_tags(self, tags: list[str]) -> list[MemoryEntry]:
        all_entries = self.list_all()
        tag_set = set(tags)
        return [e for e in all_entries if tag_set & set(e.tags)]

    def count(self) -> int:
        row = self._conn.execute("SELECT COUNT(*) FROM memories").fetchone()
        return row[0]
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/memory/test_store.py -v`
Expected: 12 passed

- [ ] **Step 5: Commit**

```bash
git add src/hal/memory/store.py tests/memory/test_store.py
git commit -m "feat: add MemoryStore with InMemory and SQLite backends"
```

---

## Task 3: Tag-Based Retrieval

**Files:**
- Create: `src/hal/memory/retrieval.py`
- Create: `tests/memory/test_retrieval.py`

- [ ] **Step 1: Write the failing tests**

`tests/memory/test_retrieval.py`:
```python
import pytest

from hal.memory.models import MemoryEntry
from hal.memory.retrieval import TagRetriever
from hal.memory.store import InMemoryMemoryStore


def _make_entry(entry_id: str, tags: list[str], confidence: float = 0.7) -> MemoryEntry:
    return MemoryEntry(
        entry_id=entry_id,
        content=f"Content for {entry_id}",
        summary=f"Summary {entry_id}",
        tags=tags,
        confidence=confidence,
        source_task="t",
        trigger="test",
    )


@pytest.fixture
def store() -> InMemoryMemoryStore:
    s = InMemoryMemoryStore()
    s.save(_make_entry("m-1", ["ios", "build", "cocoapods"], confidence=0.8))
    s.save(_make_entry("m-2", ["android", "build", "gradle"], confidence=0.6))
    s.save(_make_entry("m-3", ["ios", "test", "xctest"], confidence=0.9))
    s.save(_make_entry("m-4", ["design", "api"], confidence=0.5))
    return s


def test_retrieve_by_keywords(store):
    retriever = TagRetriever(store)
    results = retriever.retrieve(keywords=["ios", "build"], top_k=10)

    # m-1 matches 2 tags (ios, build), m-2 matches 1 (build), m-3 matches 1 (ios)
    assert len(results) == 3
    assert results[0].entry_id == "m-1"  # highest overlap


def test_retrieve_top_k(store):
    retriever = TagRetriever(store)
    results = retriever.retrieve(keywords=["ios", "build"], top_k=2)
    assert len(results) == 2


def test_retrieve_no_match(store):
    retriever = TagRetriever(store)
    results = retriever.retrieve(keywords=["python", "flask"], top_k=10)
    assert len(results) == 0


def test_retrieve_updates_last_used(store):
    retriever = TagRetriever(store)
    results = retriever.retrieve(keywords=["ios"], top_k=10)

    for entry in results:
        stored = store.get(entry.entry_id)
        assert stored.last_used is not None


def test_retrieve_returns_summaries(store):
    retriever = TagRetriever(store)
    summaries = retriever.retrieve_summaries(keywords=["ios"], top_k=10)
    assert all(isinstance(s, str) for s in summaries)
    assert len(summaries) == 2
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/memory/test_retrieval.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`src/hal/memory/retrieval.py`:
```python
"""Tag-based memory retrieval — V1 retrieval using tag overlap scoring."""

from datetime import datetime, timezone

from hal.memory.models import MemoryEntry
from hal.memory.store import MemoryStore


class TagRetriever:
    """Retrieve memories by tag overlap with task keywords."""

    def __init__(self, store: MemoryStore) -> None:
        self._store = store

    def retrieve(self, keywords: list[str], top_k: int = 10) -> list[MemoryEntry]:
        """Find top-K memories by tag overlap score.

        Score = number of matching tags × confidence.
        Updates last_used timestamp on retrieved entries.
        """
        keyword_set = set(keywords)
        all_entries = self._store.list_all()

        scored: list[tuple[float, MemoryEntry]] = []
        for entry in all_entries:
            overlap = len(keyword_set & set(entry.tags))
            if overlap > 0:
                score = overlap * entry.confidence
                scored.append((score, entry))

        scored.sort(key=lambda x: x[0], reverse=True)
        results = [entry for _, entry in scored[:top_k]]

        # Update last_used
        now = datetime.now(timezone.utc).isoformat()
        for entry in results:
            entry.last_used = now
            self._store.save(entry)

        return results

    def retrieve_summaries(self, keywords: list[str], top_k: int = 10) -> list[str]:
        """Retrieve memory summaries for system prompt injection."""
        entries = self.retrieve(keywords=keywords, top_k=top_k)
        return [e.summary for e in entries]
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/memory/test_retrieval.py -v`
Expected: 5 passed

- [ ] **Step 5: Commit**

```bash
git add src/hal/memory/retrieval.py tests/memory/test_retrieval.py
git commit -m "feat: add tag-based memory retrieval with scoring"
```

---

## Task 4: Memory Hygiene

**Files:**
- Create: `src/hal/memory/hygiene.py`
- Create: `tests/memory/test_hygiene.py`

- [ ] **Step 1: Write the failing tests**

`tests/memory/test_hygiene.py`:
```python
import pytest
from datetime import datetime, timedelta, timezone

from hal.memory.hygiene import (
    apply_confidence_decay,
    cap_initial_confidence,
    consolidate_or_store,
    evict_lowest,
)
from hal.memory.models import MemoryEntry
from hal.memory.store import InMemoryMemoryStore


def _make_entry(
    entry_id: str = "m-1",
    tags: list[str] | None = None,
    confidence: float = 0.7,
    last_used: str | None = None,
) -> MemoryEntry:
    return MemoryEntry(
        entry_id=entry_id,
        content="Test",
        summary="Test",
        tags=tags or ["test"],
        confidence=confidence,
        source_task="t",
        trigger="test",
        last_used=last_used,
    )


def test_confidence_decay_recent():
    """Memory used today — no decay."""
    now = datetime.now(timezone.utc).isoformat()
    entry = _make_entry(confidence=0.8, last_used=now)
    effective = apply_confidence_decay(entry, decay_factor=0.99)
    assert abs(effective - 0.8) < 0.01


def test_confidence_decay_60_days():
    """Memory unused for 60 days decays to ~55%."""
    past = (datetime.now(timezone.utc) - timedelta(days=60)).isoformat()
    entry = _make_entry(confidence=0.8, last_used=past)
    effective = apply_confidence_decay(entry, decay_factor=0.99)
    assert 0.43 < effective < 0.46  # 0.8 * 0.99^60 ≈ 0.44


def test_confidence_decay_never_used():
    """Never used → decay from created_at."""
    past = (datetime.now(timezone.utc) - timedelta(days=30)).isoformat()
    entry = _make_entry(confidence=0.7, last_used=None)
    entry.created_at = past
    effective = apply_confidence_decay(entry, decay_factor=0.99)
    assert effective < 0.7


def test_cap_initial_confidence_driver():
    """Driver-reported lessons capped at 0.7."""
    assert cap_initial_confidence(0.95, is_leader_shared=False) == 0.7
    assert cap_initial_confidence(0.5, is_leader_shared=False) == 0.5


def test_cap_initial_confidence_leader():
    """Leader shared lessons capped at 0.9."""
    assert cap_initial_confidence(0.95, is_leader_shared=True) == 0.9
    assert cap_initial_confidence(0.5, is_leader_shared=True) == 0.5


def test_consolidate_merges_similar():
    store = InMemoryMemoryStore()
    existing = _make_entry("m-1", tags=["ios", "build"], confidence=0.6)
    existing.times_validated = 1
    store.save(existing)

    new_entry = _make_entry("m-new", tags=["ios", "build"], confidence=0.7)

    result = consolidate_or_store(store, new_entry, tag_overlap_threshold=0.5)

    assert result == "merged"
    merged = store.get("m-1")
    assert merged.confidence == 0.7  # higher confidence kept
    assert merged.times_validated == 2  # incremented


def test_consolidate_stores_new():
    store = InMemoryMemoryStore()
    store.save(_make_entry("m-1", tags=["android", "gradle"]))

    new_entry = _make_entry("m-new", tags=["ios", "xcode"])

    result = consolidate_or_store(store, new_entry, tag_overlap_threshold=0.5)

    assert result == "stored"
    assert store.get("m-new") is not None
    assert store.count() == 2


def test_evict_lowest():
    store = InMemoryMemoryStore()
    now = datetime.now(timezone.utc).isoformat()
    store.save(_make_entry("m-1", confidence=0.3, last_used=now))
    store.save(_make_entry("m-2", confidence=0.9, last_used=now))
    store.save(_make_entry("m-3", confidence=0.5, last_used=now))

    evict_lowest(store, max_entries=2, decay_factor=0.99)

    assert store.count() == 2
    assert store.get("m-1") is None  # lowest confidence evicted
    assert store.get("m-2") is not None
    assert store.get("m-3") is not None


def test_confidence_cap_max():
    """Validation can raise confidence, but not above 0.9."""
    store = InMemoryMemoryStore()
    existing = _make_entry("m-1", confidence=0.85)
    existing.times_validated = 5
    store.save(existing)

    new_entry = _make_entry("m-new", tags=["test"], confidence=0.7)
    consolidate_or_store(store, new_entry, tag_overlap_threshold=0.5)

    merged = store.get("m-1")
    assert merged.confidence <= 0.9
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/memory/test_hygiene.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`src/hal/memory/hygiene.py`:
```python
"""Memory hygiene — confidence decay, consolidation, eviction, and caps.

- Decay is lazy (computed at retrieval time).
- Consolidation runs on write (merge similar entries).
- Eviction enforces hard caps via score-based LRU.
"""

from datetime import datetime, timezone

from hal.memory.models import MemoryEntry
from hal.memory.store import MemoryStore

CONFIDENCE_CAP = 0.9
DRIVER_CONFIDENCE_CAP = 0.7
LEADER_SHARED_CONFIDENCE_CAP = 0.9


def apply_confidence_decay(
    entry: MemoryEntry, decay_factor: float = 0.99
) -> float:
    """Compute effective confidence after time-based decay.

    effective_confidence = confidence × decay_factor ^ days_since_last_used
    """
    reference = entry.last_used or entry.created_at
    if reference is None:
        return entry.confidence

    ref_dt = datetime.fromisoformat(reference)
    days = (datetime.now(timezone.utc) - ref_dt).total_seconds() / 86400.0
    return entry.confidence * (decay_factor ** max(0, days))


def cap_initial_confidence(confidence: float, *, is_leader_shared: bool) -> float:
    """Apply initial confidence cap.

    Driver-reported: capped at 0.7.
    Leader shared: capped at 0.9.
    """
    cap = LEADER_SHARED_CONFIDENCE_CAP if is_leader_shared else DRIVER_CONFIDENCE_CAP
    return min(confidence, cap)


def consolidate_or_store(
    store: MemoryStore,
    new_entry: MemoryEntry,
    tag_overlap_threshold: float = 0.5,
) -> str:
    """Check for similar existing entries; merge or store new.

    Returns "merged" if consolidated with existing, "stored" if added as new.
    """
    new_tags = set(new_entry.tags)
    if not new_tags:
        store.save(new_entry)
        return "stored"

    for existing in store.list_all():
        existing_tags = set(existing.tags)
        if not existing_tags:
            continue
        overlap = len(new_tags & existing_tags) / max(len(new_tags), len(existing_tags))
        if overlap >= tag_overlap_threshold:
            # Merge: keep higher confidence, more detailed content, increment validation
            existing.confidence = min(
                max(existing.confidence, new_entry.confidence),
                CONFIDENCE_CAP,
            )
            existing.times_validated += 1
            if len(new_entry.content) > len(existing.content):
                existing.content = new_entry.content
                existing.summary = new_entry.summary
            store.save(existing)
            return "merged"

    store.save(new_entry)
    return "stored"


def evict_lowest(
    store: MemoryStore,
    max_entries: int,
    decay_factor: float = 0.99,
) -> None:
    """Evict lowest-scoring entries if count exceeds max_entries.

    Score = effective_confidence (with decay applied).
    """
    if store.count() <= max_entries:
        return

    entries = store.list_all()
    scored = [(apply_confidence_decay(e, decay_factor), e) for e in entries]
    scored.sort(key=lambda x: x[0])

    to_remove = len(entries) - max_entries
    for i in range(to_remove):
        store.delete(scored[i][1].entry_id)
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/memory/test_hygiene.py -v`
Expected: 10 passed

- [ ] **Step 5: Commit**

```bash
git add src/hal/memory/hygiene.py tests/memory/test_hygiene.py
git commit -m "feat: add memory hygiene — decay, consolidation, eviction"
```

---

## Task 5: Episodic Memory

**Files:**
- Create: `src/hal/memory/episodic.py`
- Create: `tests/memory/test_episodic.py`

- [ ] **Step 1: Write the failing tests**

`tests/memory/test_episodic.py`:
```python
import uuid

import pytest

from hal.core.driver import DriverResult, DriverStatus, Lesson, LessonScope
from hal.memory.episodic import EpisodicMemory
from hal.memory.store import InMemoryMemoryStore


@pytest.fixture
def episodic() -> EpisodicMemory:
    store = InMemoryMemoryStore()
    return EpisodicMemory(store=store, max_entries=5, decay_factor=0.99)


def test_ingest_lesson(episodic):
    lesson = Lesson(
        content="Pin CocoaPods to 1.14",
        trigger="build_failure",
        confidence=0.8,
        tags=["cocoapods", "build"],
        source_task="task-1",
    )
    episodic.ingest_lesson(lesson)

    entries = episodic.store.list_all()
    assert len(entries) == 1
    assert entries[0].confidence == 0.7  # capped at DRIVER_CONFIDENCE_CAP
    assert "CocoaPods" in entries[0].content


def test_ingest_lesson_high_confidence_capped(episodic):
    lesson = Lesson(
        content="Important thing",
        trigger="test",
        confidence=0.95,
        tags=["test"],
        source_task="t",
    )
    episodic.ingest_lesson(lesson)

    entries = episodic.store.list_all()
    assert entries[0].confidence == 0.7  # capped


def test_infer_from_failed_result(episodic):
    result = DriverResult(
        status=DriverStatus.FAILED,
        messages=[],
        steps_taken=5,
        token_usage=500,
        summary="Build failed: missing dependency libssl",
    )
    episodic.infer_from_result(result, task_id="task-2")

    entries = episodic.store.list_all()
    assert len(entries) == 1
    assert "missing dependency" in entries[0].content
    assert entries[0].trigger == "task_failed"


def test_infer_from_budget_exhausted(episodic):
    result = DriverResult(
        status=DriverStatus.BUDGET_EXHAUSTED,
        messages=[],
        steps_taken=50,
        token_usage=5000,
    )
    episodic.infer_from_result(result, task_id="task-3")

    entries = episodic.store.list_all()
    assert len(entries) == 1
    assert entries[0].trigger == "budget_exhausted"


def test_no_inference_from_completed(episodic):
    result = DriverResult(
        status=DriverStatus.COMPLETED,
        messages=[],
        steps_taken=3,
        token_usage=300,
    )
    episodic.infer_from_result(result, task_id="task-ok")
    assert episodic.store.count() == 0


def test_eviction_on_max_entries(episodic):
    for i in range(7):
        lesson = Lesson(
            content=f"Lesson {i}",
            trigger="test",
            confidence=0.5 + i * 0.05,
            tags=[f"tag-{i}"],
            source_task=f"task-{i}",
        )
        episodic.ingest_lesson(lesson)

    # max_entries=5, so 2 should be evicted
    assert episodic.store.count() == 5


def test_retrieve_memories(episodic):
    for tag in ["ios", "android", "web"]:
        lesson = Lesson(
            content=f"{tag} lesson",
            trigger="test",
            confidence=0.7,
            tags=[tag, "build"],
            source_task="t",
        )
        episodic.ingest_lesson(lesson)

    summaries = episodic.retrieve(keywords=["ios", "build"], top_k=5)
    assert len(summaries) >= 1
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/memory/test_episodic.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`src/hal/memory/episodic.py`:
```python
"""Episodic Memory — per-instance lessons learned from past tasks."""

import uuid

from hal.core.driver import DriverResult, DriverStatus, Lesson
from hal.memory.hygiene import cap_initial_confidence, consolidate_or_store, evict_lowest
from hal.memory.models import MemoryEntry
from hal.memory.retrieval import TagRetriever
from hal.memory.store import MemoryStore


class EpisodicMemory:
    """Per-instance memory that accumulates lessons from task execution."""

    def __init__(
        self,
        store: MemoryStore,
        max_entries: int = 200,
        decay_factor: float = 0.99,
    ) -> None:
        self.store = store
        self._max_entries = max_entries
        self._decay_factor = decay_factor
        self._retriever = TagRetriever(store)

    def ingest_lesson(self, lesson: Lesson) -> None:
        """Store a Driver-reported lesson with confidence capping and consolidation."""
        capped_confidence = cap_initial_confidence(
            lesson.confidence, is_leader_shared=False,
        )

        entry = MemoryEntry(
            entry_id=str(uuid.uuid4()),
            content=lesson.content,
            summary=lesson.content[:100],
            tags=lesson.tags,
            confidence=capped_confidence,
            source_task=lesson.source_task or "",
            trigger=lesson.trigger,
        )

        consolidate_or_store(self.store, entry)
        evict_lowest(self.store, self._max_entries, self._decay_factor)

    def infer_from_result(self, result: DriverResult, task_id: str) -> None:
        """Generate automatic lessons from DriverResult status."""
        if result.status == DriverStatus.FAILED and result.summary:
            entry = MemoryEntry(
                entry_id=str(uuid.uuid4()),
                content=result.summary,
                summary=result.summary[:100],
                tags=["failure"],
                confidence=cap_initial_confidence(0.7, is_leader_shared=False),
                source_task=task_id,
                trigger="task_failed",
            )
            consolidate_or_store(self.store, entry)
            evict_lowest(self.store, self._max_entries, self._decay_factor)

        elif result.status == DriverStatus.BUDGET_EXHAUSTED:
            entry = MemoryEntry(
                entry_id=str(uuid.uuid4()),
                content=f"Task {task_id} exhausted budget ({result.steps_taken} steps, "
                        f"{result.token_usage} tokens). Consider breaking into sub-tasks.",
                summary=f"Budget exhausted on {task_id}",
                tags=["budget", "planning"],
                confidence=cap_initial_confidence(0.7, is_leader_shared=False),
                source_task=task_id,
                trigger="budget_exhausted",
            )
            consolidate_or_store(self.store, entry)
            evict_lowest(self.store, self._max_entries, self._decay_factor)

    def retrieve(self, keywords: list[str], top_k: int = 10) -> list[str]:
        """Retrieve memory summaries for system prompt injection."""
        return self._retriever.retrieve_summaries(keywords=keywords, top_k=top_k)
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/memory/test_episodic.py -v`
Expected: 7 passed

- [ ] **Step 5: Commit**

```bash
git add src/hal/memory/episodic.py tests/memory/test_episodic.py
git commit -m "feat: add EpisodicMemory with lesson ingestion and auto-inference"
```

---

## Task 6: Shared Memory

**Files:**
- Create: `src/hal/memory/shared.py`
- Create: `tests/memory/test_shared.py`

- [ ] **Step 1: Write the failing tests**

`tests/memory/test_shared.py`:
```python
import pytest

from hal.core.driver import Lesson, LessonScope
from hal.memory.shared import SharedMemory
from hal.memory.store import InMemoryMemoryStore


@pytest.fixture
def shared() -> SharedMemory:
    store = InMemoryMemoryStore()
    return SharedMemory(store=store, max_entries=10, decay_factor=0.99, write_policy="auto")


def test_write_from_leader(shared):
    lesson = Lesson(
        content="iOS and Android must use same token refresh",
        trigger="design_decision",
        confidence=0.95,
        tags=["login", "token", "cross-platform"],
        source_task="design-auth",
        scope=LessonScope.SHARED,
    )
    shared.write_from_leader(lesson, leader_id="pm-01")

    entries = shared.store.list_all()
    assert len(entries) == 1
    assert entries[0].confidence == 0.9  # leader cap
    assert entries[0].created_by == "pm-01"


def test_write_blocked_for_non_leader(shared):
    """Member agents cannot write to shared memory via write_from_leader."""
    # SharedMemory.write_from_leader doesn't enforce caller identity —
    # the enforcement is at the routing level. But we test the
    # downgrade_member_lesson method.
    lesson = Lesson(
        content="Member lesson",
        trigger="test",
        confidence=0.7,
        tags=["test"],
        source_task="t",
        scope=LessonScope.SHARED,
    )
    result = shared.downgrade_member_lesson(lesson)
    assert result.scope == LessonScope.EPISODIC  # downgraded


def test_retrieve(shared):
    lesson = Lesson(
        content="Use v2 API for all token endpoints",
        trigger="decision",
        confidence=0.9,
        tags=["api", "token"],
        source_task="t",
        scope=LessonScope.SHARED,
    )
    shared.write_from_leader(lesson, leader_id="pm-01")

    summaries = shared.retrieve(keywords=["api", "token"], top_k=5)
    assert len(summaries) >= 1


def test_write_policy_human_confirm(shared):
    shared._write_policy = "human_confirm"

    lesson = Lesson(
        content="Needs human approval",
        trigger="decision",
        confidence=0.9,
        tags=["policy"],
        source_task="t",
        scope=LessonScope.SHARED,
    )
    result = shared.write_from_leader(lesson, leader_id="pm-01")
    assert result == "pending_human_approval"
    assert shared.store.count() == 0  # not stored yet
    assert len(shared.pending_approvals) == 1


def test_approve_pending(shared):
    shared._write_policy = "human_confirm"

    lesson = Lesson(
        content="Approved content",
        trigger="decision",
        confidence=0.9,
        tags=["approved"],
        source_task="t",
        scope=LessonScope.SHARED,
    )
    shared.write_from_leader(lesson, leader_id="pm-01")

    shared.approve_pending(0)
    assert shared.store.count() == 1
    assert len(shared.pending_approvals) == 0
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/memory/test_shared.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`src/hal/memory/shared.py`:
```python
"""Shared Memory — cluster-wide knowledge with Leader-only write access."""

import uuid

from hal.core.driver import Lesson, LessonScope
from hal.memory.hygiene import cap_initial_confidence, consolidate_or_store, evict_lowest
from hal.memory.models import MemoryEntry
from hal.memory.retrieval import TagRetriever
from hal.memory.store import MemoryStore


class SharedMemory:
    """Cluster-wide memory. Only Leader Agent can write."""

    def __init__(
        self,
        store: MemoryStore,
        max_entries: int = 500,
        decay_factor: float = 0.99,
        write_policy: str = "auto",  # "auto" or "human_confirm"
    ) -> None:
        self.store = store
        self._max_entries = max_entries
        self._decay_factor = decay_factor
        self._write_policy = write_policy
        self._retriever = TagRetriever(store)
        self.pending_approvals: list[MemoryEntry] = []

    def write_from_leader(self, lesson: Lesson, leader_id: str) -> str:
        """Write a shared lesson from Leader Agent.

        Returns "stored", "merged", or "pending_human_approval".
        """
        capped_confidence = cap_initial_confidence(
            lesson.confidence, is_leader_shared=True,
        )

        entry = MemoryEntry(
            entry_id=str(uuid.uuid4()),
            content=lesson.content,
            summary=lesson.content[:100],
            tags=lesson.tags,
            confidence=capped_confidence,
            source_task=lesson.source_task or "",
            trigger=lesson.trigger,
            created_by=leader_id,
        )

        if self._write_policy == "human_confirm":
            self.pending_approvals.append(entry)
            return "pending_human_approval"

        result = consolidate_or_store(self.store, entry)
        evict_lowest(self.store, self._max_entries, self._decay_factor)
        return result

    def approve_pending(self, index: int) -> None:
        """Approve a pending entry (for human_confirm policy)."""
        entry = self.pending_approvals.pop(index)
        consolidate_or_store(self.store, entry)
        evict_lowest(self.store, self._max_entries, self._decay_factor)

    @staticmethod
    def downgrade_member_lesson(lesson: Lesson) -> Lesson:
        """Downgrade a Member Agent's shared-scope lesson to episodic."""
        return Lesson(
            content=lesson.content,
            trigger=lesson.trigger,
            confidence=lesson.confidence,
            tags=lesson.tags,
            source_task=lesson.source_task,
            scope=LessonScope.EPISODIC,
        )

    def retrieve(self, keywords: list[str], top_k: int = 10) -> list[str]:
        """Retrieve shared memory summaries for system prompt injection."""
        return self._retriever.retrieve_summaries(keywords=keywords, top_k=top_k)
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/memory/test_shared.py -v`
Expected: 5 passed

- [ ] **Step 5: Commit**

```bash
git add src/hal/memory/shared.py tests/memory/test_shared.py
git commit -m "feat: add SharedMemory with Leader-only writes and write policy"
```

---

## Task 7: Integration with TaskRunner

**Files:**
- Create: `tests/memory/test_memory_integration.py`

- [ ] **Step 1: Write the integration test**

`tests/memory/test_memory_integration.py`:
```python
"""Integration: Driver lessons → Episodic/Shared routing → retrieval → prompt injection."""

import pytest

from hal.core.driver import DriverResult, DriverStatus, Lesson, LessonScope
from hal.memory.episodic import EpisodicMemory
from hal.memory.shared import SharedMemory
from hal.memory.store import InMemoryMemoryStore


def _route_lessons(
    lessons: list[Lesson],
    episodic: EpisodicMemory,
    shared: SharedMemory,
    is_leader: bool,
    leader_id: str = "pm-01",
) -> None:
    """Simulate TaskRunner's lesson routing logic."""
    for lesson in lessons:
        if lesson.scope == LessonScope.SHARED and is_leader:
            shared.write_from_leader(lesson, leader_id)
        else:
            if lesson.scope == LessonScope.SHARED and not is_leader:
                lesson = shared.downgrade_member_lesson(lesson)
            episodic.ingest_lesson(lesson)


def test_lesson_routing_member_agent():
    """Member Agent: shared-scope lessons downgraded to episodic."""
    episodic = EpisodicMemory(InMemoryMemoryStore(), max_entries=100)
    shared = SharedMemory(InMemoryMemoryStore(), max_entries=100)

    lessons = [
        Lesson(content="Episodic lesson", trigger="test", confidence=0.7,
               tags=["ios"], source_task="t-1", scope=LessonScope.EPISODIC),
        Lesson(content="Should be downgraded", trigger="test", confidence=0.7,
               tags=["cross-platform"], source_task="t-1", scope=LessonScope.SHARED),
    ]

    _route_lessons(lessons, episodic, shared, is_leader=False)

    assert episodic.store.count() == 2  # both stored in episodic
    assert shared.store.count() == 0    # nothing in shared


def test_lesson_routing_leader_agent():
    """Leader Agent: shared-scope lessons go to shared memory."""
    episodic = EpisodicMemory(InMemoryMemoryStore(), max_entries=100)
    shared = SharedMemory(InMemoryMemoryStore(), max_entries=100)

    lessons = [
        Lesson(content="Episodic lesson", trigger="test", confidence=0.7,
               tags=["planning"], source_task="t-1", scope=LessonScope.EPISODIC),
        Lesson(content="Shared knowledge", trigger="decision", confidence=0.9,
               tags=["api", "standard"], source_task="t-1", scope=LessonScope.SHARED),
    ]

    _route_lessons(lessons, episodic, shared, is_leader=True)

    assert episodic.store.count() == 1  # episodic-scoped only
    assert shared.store.count() == 1    # shared-scoped
    assert shared.store.list_all()[0].created_by == "pm-01"


def test_retrieval_for_prompt_injection():
    """Retrieve from both episodic and shared for combined prompt injection."""
    episodic = EpisodicMemory(InMemoryMemoryStore(), max_entries=100)
    shared = SharedMemory(InMemoryMemoryStore(), max_entries=100)

    episodic.ingest_lesson(Lesson(
        content="CocoaPods 1.15 breaks Xcode 16",
        trigger="build_failure",
        confidence=0.7,
        tags=["cocoapods", "build"],
        source_task="t-1",
    ))
    shared.write_from_leader(
        Lesson(
            content="Use v2 token API",
            trigger="decision",
            confidence=0.9,
            tags=["api", "token"],
            source_task="t-2",
            scope=LessonScope.SHARED,
        ),
        leader_id="pm-01",
    )

    # Combined retrieval
    episodic_mems = episodic.retrieve(keywords=["cocoapods", "build"], top_k=5)
    shared_mems = shared.retrieve(keywords=["api", "token"], top_k=5)

    all_memories = episodic_mems + shared_mems
    assert len(all_memories) >= 2
```

- [ ] **Step 2: Run integration test**

Run: `pytest tests/memory/test_memory_integration.py -v`
Expected: 3 passed

- [ ] **Step 3: Run full test suite**

Run: `pytest tests/ -v`
Expected: All tests pass

- [ ] **Step 4: Commit**

```bash
git add tests/memory/test_memory_integration.py
git commit -m "test: add Memory System integration tests with lesson routing"
```

---

## Summary

| Task | Component | Test count (est.) |
|------|-----------|-------------------|
| 1 | Memory Entry Model | 3 |
| 2 | Memory Store (InMemory + SQLite) | 12 |
| 3 | Tag-Based Retrieval | 5 |
| 4 | Memory Hygiene | 10 |
| 5 | Episodic Memory | 7 |
| 6 | Shared Memory | 5 |
| 7 | Integration | 3 |
| **Total** | | **~45** |
