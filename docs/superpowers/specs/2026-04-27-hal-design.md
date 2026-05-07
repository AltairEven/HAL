# HAL Framework — Design Specification

A general-purpose, highly decoupled agent coordination platform for constructing role-specialized AI agent teams.

## Overview

HAL (inspired by HAL 9000 from *2001: A Space Odyssey*) is a framework for building **specialist AI agents** — agents that behave like dedicated professionals with clear expertise boundaries, not generalist chatbots. Multiple HAL instances can form a **cluster**, collaborating through structured protocols like a real team.

HAL does NOT implement the agent's internal reasoning loop (LLM calls, tool execution dispatch, chain-of-thought). That is delegated to an **AgentDriver** — a pluggable adapter for external agent management tools (e.g., Claude Agent SDK, LangGraph, custom implementations). HAL focuses on what sits above and around the individual agent: coordination, identity, tools, memory, and lifecycle management.

### Core Philosophy

- **One architecture, multiple roles** — The framework is universal; specialization comes from configuration, not code.
- **Specialist, not generalist** — Each agent instance is a dedicated professional (e.g., "Senior iOS Developer"), with enforced expertise boundaries.
- **Human control without Human bottleneck** — Human makes critical decisions; agents handle everything within their granted authority.
- **Distributed by design** — Agents may run on different machines, communicating through serializable protocols.
- **Execution-agnostic** — HAL coordinates; the Driver executes. Swap the Driver, keep the coordination.

### Design Principles

| Principle | Meaning |
|-----------|---------|
| Engineering Safety | Role boundaries enforced. Tools sandboxed. Actions auditable. |
| AI Autonomy | Within granted scope, agents act freely without Human bottleneck. |
| Human Control | HITL limited to agent escalation and high-risk tool confirmation. Safety boundaries enforced by external systems (git ACL, CI pipelines, deploy workflows), not duplicated by the framework. |

### Responsibility Split

| HAL owns | AgentDriver owns |
|---|---|
| Multi-agent coordination (Leader Agent/Member Agent, task routing, escalation) | LLM API calls (model selection, prompt formatting, function calling) |
| Inter-agent communication (HAL Bus + Artifacts) | Internal reasoning loop (reason-act-reflect, chain-of-thought) |
| Identity & Role (role profiles, tool configuration, boundaries) | Tool execution dispatch (calling tool.execute() based on LLM decisions) |
| Tool registry and loading (defining WHAT tools are available) | Completion and escalation decisions |
| Working Memory (conversation history, context management) | Working Memory truncation/compression (context window management) |
| | Enforcing per-call step/cost limits (from remaining budget passed by HAL) |
| Checkpoint / resume (task-level persistence) | |
| Safeguard accounting (cumulative step/cost tracking, hard cost ceiling) | |

---

## Five-Layer Architecture

```
+------------------------------------------+
|        Layer 5: Cluster Management       |
|   Leader collect-decide / HITL routing   |
+------------------------------------------+
                    |
+------------------------------------------+
|        Layer 4: Task Runner              |
|   AgentDriver + Checkpoint + Safeguards  |
+------------------------------------------+
                    |
+------------------------------------------+
|        Layer 3: Identity & Role          |
|   Role / Boundaries / Knowledge          |
+------------------------------------------+
                    |
+------------------------------------------+
|        Layer 2: Tool Arsenal             |
|   Tool protocol / Loader / Sandbox       |
+------------------------------------------+
                    |
+------------------------------------------+
|        Layer 1: Core Kernel              |
|   Config / AsyncEventBus / Logger /      |
|   PluginRegistry / AgentDriver / HALBus  |
+------------------------------------------+
```

---

## Layer 1: Core Kernel

Foundation infrastructure used by all upper layers.

### Components

- **Config Loader** — Loads framework and role configuration (YAML/JSON). Supports environment variable overrides and layered config merging.
- **AsyncEventBus** — Internal async publish/subscribe mechanism for decoupling layers. Events include `tool.called`, `checkpoint.saved`, `artifact.published`, and task lifecycle events (`task.started`, `task.completed`, `task.failed`, `task.suspended`, `task.resumed`). Supports wildcard subscriptions. A failing handler is logged but does not prevent other handlers from running.
- **Logger** — Structured JSON logging with trace context. Every step produces a traceable log entry with agent ID, task/step context, trace_id, and arbitrary key-value data.
- **Plugin Registry** — Central registry where tools, roles, storage backends, and transport adapters register as named plugins under a category.
- **Error Handler** — Unified error classification: Recoverable (retry), Unrecoverable (fatal), EscalateToHuman (needs Human decision).

### Tracing

Every Human request generates a `trace_id` (UUID) at the ClusterRouter. The trace_id propagates through the full request lifecycle:

- ClusterRouter stamps the trace_id on the initial Artifact sent to Leader Agent
- Leader Agent's cluster management tools (assign_task, collect_opinions, etc.) inherit the trace_id onto all sub-task Artifacts
- Each AgentService, TaskRunner, and Tool call includes the trace_id in log entries
- All Artifacts in a request chain share the same trace_id

This enables full-chain tracing: given a trace_id, all log entries and Artifacts related to a single Human request can be retrieved and correlated across all agents in the cluster.

### AgentDriver

The core abstraction for delegating agent execution to external SDKs. HAL calls the Driver with context (conversation history, tools, system prompt) and constraints (step limit, cost budget). The Driver runs its internal LLM loop and returns a result.

**Driver input:**
- Conversation history from Working Memory
- Tool descriptions (JSON Schema for LLM) + tool executors (SandboxedTool wrappers for execution)
- System prompt (built from role profile + injected memories)
- Constraints: max steps, cost budget
- Optional: cancellation token (see Key design decisions)

**Driver output (DriverResult):**
- **Status** — one of: COMPLETED, ESCALATED, SUSPENDED, BUDGET_EXHAUSTED, FAILED
- Updated conversation history (to be stored back into Working Memory)
- Steps taken, token usage, and cost_usd (USD cost self-reported by Driver, used by Safeguards for budget tracking)
- Context-specific data: completion summary, escalation reason, output artifacts, Driver-reported lessons (for Episodic/Shared Memory), etc.
- **suspend_context** (required when status is SUSPENDED) — structured data that the Driver transparently passes through from the PENDING tool result:
  - `resume_event_id`: Optional[str] — external event ID to wait for (from tool PENDING result)
  - `confirmation_request`: Optional[dict] — high-risk tool approval request (from SandboxedTool)
  - TaskRunner uses this to determine post-suspension action: publish `confirmation_request` Artifact, or record `resume_event_id` and wait. When SUSPENDED is caused by cancellation_token (preemption), suspend_context is absent — AgentService already knows the cause since it initiated the cancellation.

**Key design decisions:**

- **No LLMProvider in HAL.** The Driver handles LLM selection, API calls, and prompt formatting internally. HAL does not need to know which model is used.
- **Constraints as parameters.** max_steps and cost_budget are passed to the Driver. The Driver is responsible for respecting these limits and returning BUDGET_EXHAUSTED when reached. HAL trusts the Driver's self-reported token usage for Safeguard tracking — no independent cost verification is performed. As a failsafe, HAL enforces a hard cost ceiling (`max_cost_per_task`, configurable in cluster config, default: 50.0 USD). If the Driver's cumulative self-reported cost exceeds this ceiling, TaskRunner immediately FAILs the task (no escalation). This guards against buggy Driver implementations that ignore cost_budget constraints.
- **Optional cancellation token.** Driver input includes an optional `cancellation_token` — a lightweight async flag that HAL can set between Driver steps. If the Driver supports it, it checks the token between steps and returns early with SUSPENDED status when cancellation is requested, preserving the conversation history for later resumption. This enables Human overrides (pause/abort) and Leader-directed preemption to take effect during a long Driver run, rather than waiting for the entire run to complete. Drivers that do not support cancellation simply ignore the token — override takes effect after the Driver returns.
- **Messages are opaque to HAL.** HAL stores and passes messages without parsing their internal structure. The format is determined by the Driver (e.g., Claude format, OpenAI format). A checkpoint is only resumable by the same Driver type that created it. HAL adds its own messages (task description, event data) in a simple role+content format that any Driver can consume alongside its native messages.
- **Tool executors are SandboxedTool wrappers.** HAL does NOT pass raw Tool objects to the Driver. Instead, it wraps each Tool in a SandboxedTool that validates parameters, enforces risk_level confirmation flows, and emits audit events — all transparently. The Driver calls execute() on these wrappers without knowing about the sandbox. See Layer 2 for details.
- **PENDING tool results.** When a tool returns PENDING status, the Driver should stop its internal loop and return with SUSPENDED status. HAL handles checkpoint and resumption.
- **Driver implementations are external.** HAL provides a MockDriver for testing. Real implementations (e.g., wrapping Claude Agent SDK) are separate packages.

### ClaudeCodeDriver — V1 Reference Implementation

The first production AgentDriver, wrapping Claude Code's headless/SDK mode. This section captures the key integration questions and design decisions specific to this Driver.

**Open integration questions (to be resolved during implementation):**

- **Tool coexistence:** Claude Code has built-in tools (Read/Write/Edit/Bash/Glob/Grep). HAL injects SandboxedTool wrappers via JSON Schema. The two tool sets may coexist or conflict. Resolution depends on Claude Agent SDK's tool configuration API — whether built-in tools can be selectively disabled or overridden. If not, the Driver must reconcile both sets (e.g., proxy Claude Code's built-in tools through SandboxedTool wrappers, or accept that built-in tools bypass HAL's tool layer and rely on logical isolation via role boundaries).
- **Session history interop:** Working Memory requires passing conversation history into and out of the Driver for checkpoint/resume. This depends on whether Claude Agent SDK exposes session history as a serializable object or only accepts initial prompts. If history is not extractable, the Driver must maintain its own shadow copy.
- **Cost/steps accounting:** Claude Code's internal compaction pipeline makes token consumption opaque. V1 treats cost_budget as best-effort — the Driver self-reports usage, HAL trusts the report. Precise cost tracking is deferred.
- **Cancellation support:** Whether Claude Agent SDK supports mid-execution cancellation determines preemption behavior. If unsupported, preemption falls back to soft mode (new task queued at front, starts after current Driver call completes naturally).

These questions are implementation-time concerns, not design blockers. The AgentDriver protocol is designed to accommodate varying Driver capabilities — the contract is intentionally loose on these points.

### HAL Bus

Async artifact transport between agent instances. Pluggable message queue backend.

**Core capabilities:**
- **Publish** — Send an artifact to a target agent instance
- **Subscribe** — Register an async handler for incoming artifacts
- **Wait for reply** — Publish an artifact and await a response artifact (request-reply pattern). Supports a `timeout` parameter to prevent indefinite blocking (e.g., when the target agent is suspended or preempted). Used by short-lived request-reply tools (e.g., `collect_opinions`). Note: task delegation (`assign_task` + `wait_for_tasks`) uses a framework-level suspend/resume mechanism via `parent_task_id` instead of Bus-level `wait_for_reply`, to avoid deadlocks when sub-tasks escalate.
- **Inbox query** — Retrieve pending artifacts for an instance, optionally filtered by task
- **Audit trail** — Retrieve all artifacts for a given task
- **Interceptor** — Register a predicate+handler pair. Before normal delivery, each published artifact is checked against all interceptors. If a predicate matches, the artifact is routed to the interceptor's handler instead of normal delivery. Used by ClusterRouter to intercept `escalation` and `confirmation_request` artifacts transparently — agents are unaware of the interception.

**Backends:**

| Backend | Use case | Dependencies | Status |
|---------|----------|-------------|--------|
| InMemoryBus | Single-process mode. Async, in-process. | None | Implemented |
| SQLiteBus | Multi-process on same machine. SQLite WAL file as shared message queue, polling-based. | None (Python stdlib `sqlite3`) | Design-ready, not implemented |
| RedisBus | Cross-machine distributed. Redis Streams with consumer groups. | Redis server | Design-ready, not implemented |

**Design principles:** Transport agnostic. Guaranteed delivery (at-least-once). Artifacts immutable once published. Full audit trail.

**Backpressure:** Each agent's inbox has a maximum capacity (`max_inbox_size`, default: 100), configured globally in the cluster config `bus` section. When an agent's inbox reaches the limit, new Publish calls targeting that agent are rejected — the Bus returns a FAILURE result to the sender. The sender (typically Leader Agent's LLM via `assign_task`) receives the failure and decides how to handle it (retry later, reassign to another agent, or escalate). This prevents unbounded inbox growth from slow consumers, crashed agents, or long-suspended agents.

**Idempotency requirement:** At-least-once delivery means duplicate Artifacts are possible (e.g., network retry, Bus backend redelivery). Every Artifact carries a unique `artifact_id`. AgentService MUST track processed artifact IDs and skip duplicates. This is enforced at the AgentService level, not the Bus level — the Bus guarantees delivery, consumers guarantee idempotency.

**V1 simplification:** In single-process mode with InMemoryBus, duplicate delivery does not occur (in-memory async dispatch is exactly-once). V1 implements the `artifact_id` field and basic in-memory dedup (a Set per AgentService), but does NOT persist the ID set to checkpoint or handle cross-restart dedup. This is sufficient for V1's single-process topology. Full persistence (ID set persisted alongside checkpoint, restored on process restart, lifetime bound to task lifecycle) is required for V2+ distributed backends where network retries make duplicates possible.

### Artifact Model

Every handoff between agents is packaged as an Artifact: a self-describing, serializable, immutable-after-publish deliverable.

An Artifact carries: type, sender/receiver instance IDs, task ID, parent_task_id, trace_id, summary, payload, authority, and optional fields for request-reply pattern (reply_to) and suspend/resume pattern (resume_event). Once published, content fields cannot be modified. The Artifact model is append-only — task lifecycle tracking (assign → suspend → resume → complete) is achieved by correlating multiple Artifacts sharing the same task ID, not by mutating a single Artifact.

**Task Hierarchy** — Tasks support a single-level parent-child relationship via `parent_task_id`. When Leader Agent dispatches sub-tasks via `assign_task`, each sub-task carries the Leader's current task ID as its `parent_task_id`. This enables:
- **Context continuity** — Leader's main task remains SUSPENDED while sub-tasks execute. All sub-task state changes (completion, failure, escalation) are delivered as `resume_event`s to the parent task, allowing Leader to process them within the same Working Memory context.
- **Unified event delivery** — Sub-task escalations are published as `escalation` Artifacts to the Bus. ClusterRouter intercepts them and uses `parent_task_id` to translate them into `resume_event`s on the parent task, rather than delivering them as separate tasks to Leader. This keeps Leader's Working Memory context intact across the full orchestration lifecycle.
- **Queryability** — `query_agent_tasks` returns each task's `parent_task_id`, allowing Leader to filter and correlate sub-tasks across agents. No dedicated query tool needed.

**Authority** — Every Artifact carries an `authority` field indicating the source's trust level:

| Authority | Meaning | Who can set it |
|-----------|---------|----------------|
| `MEMBER` | From a peer or subordinate agent | Default for all Agent-originated Artifacts. Enforced by Bus — Agents cannot set other values. |
| `HUMAN` | From Human | Only ClusterRouter can set this, when forwarding Human responses. |

Authority is a **framework-enforced field**, not a self-declaration. Agents publish Artifacts through AgentService, which hardcodes `authority=MEMBER`. Only ClusterRouter has the privilege to stamp `HUMAN`. This is analogous to DKIM signing — the sender cannot forge the stamp.

When TaskRunner injects an Artifact's content into the Driver's conversation, it prefixes the message based on authority:
- `HUMAN` → `[HUMAN DIRECTIVE]` — the Driver's LLM treats this as an authoritative instruction
- `MEMBER` → `[MEMBER REPORT from {sender}]` — the Driver's LLM treats this as a peer's deliverable to evaluate

This prevents a Member Agent's LLM (even if hallucinating) from impersonating Human authority — the structural authority field cannot be influenced by payload content.

| Type | Description | Typical Flow | V1 Framework Handling |
|------|-------------|-------------|----------------------|
| `task_assignment` | Sub-task description, expectations | Leader Agent → Member Agent | **Special:** AgentService queue insertion. Carries `parent_task_id` linking to Leader's current task. |
| `resume_event` | External event triggering agent resume | Webhook Adapter / HAL instance → Agent | **Special:** TaskRunner checkpoint restore |
| `escalation` | Agent self-escalation requesting help | Any Agent → Bus → ClusterRouter | **Special:** ClusterRouter routes based on `parent_task_id`: if present, translates to `resume_event` on parent task (Leader receives in same Working Memory context); if absent (Leader Agent), forwards to Human. |
| `confirmation_request` | High-risk tool requesting Human approval | Any Agent → ClusterRouter → Human | **Special:** ClusterRouter routing to Human |
| `task_status_update` | Task status change notification (e.g., preempted task auto-resumed) | AgentService → Bus → ClusterRouter | **Special:** ClusterRouter translates to `resume_event` on parent task via `parent_task_id` |
| `decision` | Approved direction, resolved conflict | Leader Agent → Member Agent | Default Bus delivery |
| `code_change` | Branch, changed files, build status, test coverage | Dev → QA or Dev → Reviewer | Default Bus delivery |
| `design_doc` | Requirements, API specs, wireframes | Leader Agent → Member Agent | Default Bus delivery |
| `test_report` | Pass/fail results, bug list, reproduction steps | QA → Dev | Default Bus delivery |
| `review_feedback` | Code review comments, approval/rejection | Reviewer → Dev | Default Bus delivery |
| `catch_up` | Degradation period log (escalations + Human responses during Leader unavailability) | ClusterRouter → Leader Agent | **V2** (requires degradation mode) |

---

## Layer 2: Tool Arsenal

Role-based dynamic tool loading with visibility control and risk-level enforcement.

### Tool Protocol

Every tool implements a unified protocol with three operations:
- **execute** — Perform the action. Async — may involve IO (file system, network, subprocess).
- **validate** — Pre-check before execution (parameter validation). Sync, local-only.
- **rollback** — Undo the action if possible. Async. Optional — tools that modify state may implement this for the Driver to call when needed.

Each tool also declares a **risk_level** (low/medium/high):
- `low` / `medium` — Informational. Used for logging and audit purposes. No behavioral impact.
- `high` — **Behavioral.** SandboxedTool automatically triggers a Human confirmation flow before execution: returns PENDING status with a `confirmation_request`, causing the Driver to SUSPEND. The request is routed to Human via ClusterRouter. The tool only executes if Human approves. This ensures that dangerous operations (deploy, delete branch, modify production config) always require Human sign-off, regardless of which agent or authority level initiated them.

### Tool Result — Three Statuses

- **SUCCESS** — Operation completed successfully. Driver continues its loop.
- **FAILURE** — Operation rejected or failed (validation error, tool execution error). Driver should re-plan or escalate.
- **PENDING** — Operation initiated but waiting for external completion (MR review, CI pipeline, another HAL instance). Carries a resume_event ID. Driver should stop and return SUSPENDED.

### SandboxedTool — Tool Wrapper

The Driver never receives raw Tool objects. HAL wraps each Tool in a SandboxedTool that transparently enforces validation, risk-level controls, and audit before/after execution. The wrapper follows the same interface as Tool, so the Driver is unaware of the wrapping.

The SandboxedTool performs four steps on each execute() call:
1. **Validate** — Call the underlying tool's validate()
2. **Risk check** — If the tool's `risk_level` is `high`, return PENDING with a `confirmation_request` instead of executing. The Agent suspends and the request is routed to Human for approval. Execution only proceeds on resume with approval.
3. **Execute** — Call the underlying tool's execute()
4. **Audit** — Emit a tool.called event via AsyncEventBus for logging

This design ensures that **all** tool calls from the Driver go through validation, risk-level enforcement, and audit, regardless of Driver implementation.

**V1 isolation strategy:** V1 enforces **logical isolation** (tool visibility control) at the framework level — agents only see tools explicitly granted by their role. **Physical isolation** (filesystem path restrictions, network restrictions) is delegated to the deployment environment: in distributed mode, agents are naturally isolated on separate machines/containers; in single-process mode, physical isolation is not enforced. Path-level sandbox rules (allow_paths, deny_paths) are a V2+ enhancement to be implemented alongside OS-level mechanisms (Docker, chroot, Seatbelt/Landlock) for multi-process and single-machine deployments.

### Three-Tier Configuration

**Tier 1: Preset toolkit (fastest)**

```yaml
toolkit: mobile_dev   # preset bundle: filesystem + git + build + search
```

**Tier 2: Additive/subtractive (common case)**

```yaml
toolkit: mobile_dev
tools_add:
  - cocoapods
tools_remove:
  - deploy
```

**Tier 3: Fully custom (V2+ — fine-grained path-level control)**

```yaml
# V2+: Path-level sandbox requires OS-level isolation support
tools:
  - name: filesystem
    permissions:
      allow_paths: ["/project/ios/*"]
      deny_paths: ["/project/android/*"]
  - name: git
    permissions:
      allow_actions: [commit, branch]
```

### Design Decisions

- **Allowlist, not blocklist** — A role only sees tools explicitly granted. Unknown tools are denied by default.
- **Logical isolation first, physical isolation by environment** — V1 enforces tool visibility (which tools an agent can see and call). Physical isolation (filesystem path restrictions) is delegated to the deployment environment and planned as a V2+ framework enhancement. See SandboxedTool V1 isolation strategy above.
- **Rollback support** — Tools that modify state may optionally implement rollback. Rollback is not triggered by HAL — it is exposed via SandboxedTool for the Driver to call when needed during its own error recovery.

---

## Layer 3: Identity & Role

Structured role profiles that make agents specialists, not generalists. A role is not just a system prompt — it is a structural constraint system that drives tool filtering, boundary enforcement, and knowledge preloading.

### Role Profile Structure

```yaml
identity:
  name: "Senior iOS Developer"
  persona: |
    You are a senior iOS developer with 8 years of experience.
    You write clean, testable Swift code following SOLID principles.
    You are meticulous about code review and documentation.

boundaries:
  can_do:
    - "Write and modify Swift / Objective-C code"
    - "Run Xcode builds and unit tests"
    - "Review iOS-related merge requests"
    - "Propose technical solutions for iOS features"
  cannot_do:
    - "Modify Android or HarmonyOS code"
    - "Make product decisions"
    - "Approve final deployment"
  escalate_when:
    - "Requirement is ambiguous or contradictory"
    - "Change impacts public API contract"
    - "Estimated effort exceeds 2 days"

knowledge:
  domains: [ios, swift, uikit, swiftui, cocoapods, xcode]
  context_docs:
    - path: "/project/docs/ios-architecture.md"
    - path: "/project/docs/coding-standards.md"

concurrency:
  max_concurrent_tasks: 1        # 最大并行任务数，默认 1（串行）。Agent 固有属性，由角色特性决定，Leader Agent 不可动态调整。

safeguards:
  max_steps: 50                  # 单任务最大步数，默认 50
  cost_budget: 5.0               # 单任务最大成本（USD），默认 5.0
  max_escalations_per_task: 3    # 单任务最大 escalation 次数，默认 3，超限自动 FAIL
  suspend_ttl: 86400             # 挂起超时（秒），默认 24 小时，超时后 SUSPENDED→escalate / ESCALATED→fail

memory:
  max_episodic: 200              # Episodic Memory 上限，默认 200
  max_shared: 500                # Shared Memory 上限，默认 500

tools:
  toolkit: mobile_dev
  tools_add:
    - cocoapods
  tools_remove:
    - deploy
```

### How Identity Shapes Behavior

1. **System Prompt Injection** — `persona` and `boundaries` are baked into the system prompt passed to the Driver, constraining the agent at the cognitive level.
2. **Tool Filtering** — Only tools explicitly configured in the role's `tools` section (toolkit + tools_add/tools_remove) are mounted. The role author is responsible for ensuring the tool configuration aligns with the `can_do` scope. There is no automatic mapping from `can_do` text to tool selection — `can_do` constrains the LLM's reasoning via system prompt, while `tools` constrains available actions via Layer 2.
3. **Boundary Enforcement** — Two layers:
   - **Cognitive level (best-effort):** `persona` and `boundaries` in system prompt constrain LLM reasoning. Cannot prevent all drift — the LLM could theoretically produce incorrect out-of-scope reasoning in text output. This is an inherent limitation of LLM-based agents.
   - **Tool visibility level (hard-enforced):** Only tools granted by the role are visible to the Driver. The LLM cannot call tools it doesn't know exist. Combined with SandboxedTool's risk_level confirmation flow, this provides the V1 enforcement layer. V2+ adds physical isolation (path-level sandbox) for defense in depth.
4. **Knowledge Preloading** — `context_docs` loaded at agent startup and appended to system prompt. `domains` influence memory retrieval priorities.

### Specialist vs Generalist

| Generalist (what we avoid) | Specialist (what HAL creates) |
|---|---|
| "I can do everything — write code, design UIs, manage projects, test, deploy..." | "I am an iOS developer. I write Swift code, run tests, and review iOS MRs. For anything else, I'll ask the right person." |
| Mediocre at everything, no clear responsibility, unpredictable behavior | Deep expertise, clear boundary, predictable and trustworthy |

---

## Layer 4: Task Runner

A thin orchestrator that delegates execution to the AgentDriver and manages the task lifecycle. HAL does not own the LLM reasoning cycle.

### Task Execution Flow

```
TaskRunner.run(task_id, task_description):

  1. Prepare context
     → Add task description to Working Memory
     → Build system prompt: static base (role persona + boundaries + context_docs from Layer 3)
       + dynamic tail (episodic/shared memories retrieved for this specific task)

  2. Call Driver
     → Pass: messages, sandboxed tools, system prompt, constraints
     → Driver runs internally (LLM loop, tool calls, etc.)
     → Driver returns DriverResult

  3. Post-execution
     → Update Working Memory with Driver's returned messages
     → Record steps and cost_usd in Safeguards
     → Store Driver-reported lessons into Episodic Memory + status-based inference
     → Save Checkpoint

  4. Handle result status
     → COMPLETED → return success with summary + artifacts
     → ESCALATED → save checkpoint (set suspended_at), route escalation, wait for response
     → SUSPENDED → save checkpoint (set suspended_at), wait for `resume_event`
     → BUDGET_EXHAUSTED → treat as escalation (request more budget)
     → FAILED → return failure with error
```

### Task Resume Flow

```
TaskRunner.resume(task_id, event_data):

  1. Restore from Checkpoint
     → Restore Working Memory snapshot
     → Restore Safeguards state (cumulative steps/cost)

  2. Prepare resumed context
     → Re-inject memories (may have new entries since last run)
     → Inject event data into conversation (external event, escalation response, etc.)

  3. Call Driver → same as run()
  4. Post-execution → same as run()
  5. Handle result → same as run()
```

### HITL — Escalation and High-Risk Confirmation

**Design principle: safety boundaries belong to external systems, not to the framework.**

HITL is limited to two scenarios:

1. **Agent self-escalation** — The Driver returns ESCALATED. TaskRunner saves checkpoint and publishes an `escalation` Artifact to the Bus. ClusterRouter intercepts and routes it: for Member Agents (Artifact carries `parent_task_id`), ClusterRouter translates it into a `resume_event` on Leader's parent task; for Leader Agent (no `parent_task_id`), ClusterRouter forwards it to Human. The agent enters ESCALATED state (similar to SUSPENDED), waiting for a response. When Leader Agent/Human responds, the agent resumes via `resume_event` with guidance as event data.
2. **High-risk tool confirmation** — SandboxedTool detects a `risk_level: high` tool call and returns PENDING with a `confirmation_request`. The Driver returns SUSPENDED (with `suspend_context.confirmation_request`). TaskRunner publishes a `confirmation_request` Artifact; ClusterRouter routes it directly to Human (bypassing Leader Agent). Human approves or rejects; the agent resumes accordingly. See Escalation Routing for the full flow.

### Safeguards

Safeguards track cumulative execution limits across multiple Driver calls (e.g., across suspend/resume cycles):
- **Max Steps** — Total step budget. Configurable per role. Default: 50.
- **Cost Budget** — Total USD budget for token consumption.
- **Max Escalations Per Task** — Maximum number of escalations allowed within a single task lifecycle. Configurable per role. Default: 3. When exceeded, the task is automatically FAILED instead of escalating again. This prevents runaway escalation loops from a misbehaving Driver or unsolvable task from flooding Leader Agent or Human.

Remaining steps and budget are passed TO the Driver as constraints. The Driver is responsible for respecting these limits. When BUDGET_EXHAUSTED is returned, TaskRunner treats it as an escalation — the Leader Agent or Human decides whether to allocate more budget.

### Checkpoint

Checkpoint includes ALL state needed for recovery, including in distributed mode where the process may restart on a different machine. Key contents:
- `schema_version` — HAL checkpoint format version. On restore, HAL validates that the checkpoint's schema_version matches the current HAL version. Mismatched checkpoints are not migrated — the task is failed immediately and escalated to Leader Agent/Human. This is a deliberate V1 simplification: checkpoint format migration adds complexity with minimal benefit while the schema is still evolving.
- Agent instance identifier
- Driver type (for compatibility — a checkpoint is only resumable by the same Driver type)
- Full Working Memory snapshot (opaque to HAL — stored as-is from the Driver)
- Safeguards state (cumulative steps used, cost used)
- Current task status (SUSPENDED, ESCALATED, etc.)
- Context-specific data (suspend event ID, escalation reason)
- `suspended_at` timestamp (when the task entered SUSPENDED or ESCALATED state)

Pluggable backends: InMemoryCheckpointStore (testing), SQLiteCheckpointStore (production).

**Suspend TTL:** Every suspended checkpoint has a configurable `suspend_ttl` (default: 24 hours, configurable per role). TaskRunner sets the `suspended_at` timestamp when transitioning to SUSPENDED or ESCALATED state. If no `resume_event` arrives within the TTL, the task is automatically transitioned: SUSPENDED tasks are escalated (published as `escalation` Artifact to Bus, carrying `parent_task_id` if present — ClusterRouter applies standard routing rules), ESCALATED tasks are failed with a timeout error. This prevents indefinite resource leakage from lost webhooks, unresponsive Humans, or abandoned tasks. Note: Tasks suspended for preemption are also checkpointed as SUSPENDED, but are tracked separately in AgentService's persisted `preempted_task_ids` list. AgentService resumes these automatically when slots become available — they are NOT subject to Suspend TTL.

TTL scanning is performed by AgentService as a periodic background task (not by ClusterRouter), since AgentService owns the checkpoint store and TaskRunner lifecycle. AgentService checks its own suspended checkpoints at a configurable interval (default: 1 minute). ClusterRouter's health monitoring is limited to Leader Agent liveness; per-agent TTL management remains an agent-level concern.

---

## Layer 5: Cluster Management

### Architecture: ClusterRouter + Leader Agent

A HAL cluster separates **infrastructure** (deterministic routing, always available) from **intelligence** (LLM-based decision-making, may fail):

| Concern | Component | Nature | Failure risk |
|---------|-----------|--------|-------------|
| Message routing, escalation queuing, health monitoring, Human interface | **ClusterRouter** | Infrastructure, no LLM | Near zero |
| Task decomposition, conflict resolution, quality review, Human communication | **Leader Agent** | LLM-based HAL instance | API timeout, hallucination, budget exhaustion |

**Scalability note:** Leader Agent processes all management decisions (task dispatch, escalation resolution, quality review) through sequential LLM calls. In a cluster with N Member Agents, concurrent task completions queue at the Leader. For typical task durations (10+ minutes per sub-task), Leader scheduling latency (~10-30s per LLM call) is negligible. This architecture is designed for teams of N≤10 agents. Larger clusters would require hierarchy extensions (sub-Leaders) not covered in V1.

This separation ensures that the cluster's "skeleton" (routing, Human interface) remains online even when the Leader Agent's LLM fails. Member Agents never contact Human directly — they communicate through the Bus, and ClusterRouter handles all Human-facing interaction transparently.

**Single-instance cluster:** A cluster with only one HAL instance is not a separate "mode" — it uses the same architecture. ClusterRouter is still present and handles Human interaction. The single instance acts as both Leader Agent and executor. Communication mechanisms are identical to multi-instance clusters.

```
Human (HTTP/WebSocket)
  │
  ▼
┌─────────────────────────────────────┐
│  ClusterRouter (infrastructure)     │
└────────────────┬────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────┐
│  HAL Bus                            │
└────────────────┬────────────────────┘
                 │
                 ▼
          HAL Instance
      (single agent, Leader role)
```

**Multi-instance cluster:** ClusterRouter is the Human-facing infrastructure layer. Leader Agent is a regular HAL instance with additional management tools. All other Member Agents are regular HAL instances.

```
Human (HTTP/WebSocket)
  │
  ▼
┌─────────────────────────────────────┐
│  ClusterRouter (infrastructure)     │  ← No LLM, deterministic, always up
│  - Human Interface (HTTP/WebSocket)  │
│  - Artifact routing rules           │
│  - Leader Agent liveness monitor    │
│  - Streaming log (WebSocket)        │
│  - Cluster status dashboard         │
└────────────────┬────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────┐
│  HAL Bus                            │
└──┬──────────┬──────────┬──────────┬─┘
   ▼          ▼          ▼          ▼
Leader     iOS Dev    Android Dev   QA
Agent      Agent      Agent        Agent
```

**All agents (including Leader Agent) are standard HAL instances.** They only interact with the Bus. They are unaware of ClusterRouter's existence.

### ClusterRouter

ClusterRouter is a **cluster-level infrastructure component**, not a HAL Agent instance. It has no LLM, no role profile, no Driver. It is a lightweight async service with deterministic routing logic.

**Responsibilities:**
- **Human interface** — Expose HTTP/WebSocket for Human to submit tasks, view status, and respond to escalations/confirmations
- **Artifact routing** — Monitor the Bus and apply routing rules (see table below)
- **Authority stamping** — Set `authority=HUMAN` on Artifacts created from Human responses. This is a privilege only ClusterRouter holds.
- **Leader Agent lifecycle** — Monitor Leader Agent liveness. If it crashes, notify Human "Leader Agent unavailable, cluster paused". V1 does not implement automatic recovery or degradation fallback — Leader failure stops the cluster. V2+ will add degradation mode (direct Human forwarding, `degradation_log`, `catch_up` Artifact for Leader recovery).
- **Streaming log** — Subscribe to AsyncEventBus events (`tool.called`, `checkpoint.saved`, `artifact.published`, `task.started`, `task.completed`, `task.failed`, `task.suspended`, `task.resumed`) and stream them to Human via WebSocket in real-time. Task lifecycle events are emitted by TaskRunner at state transitions (not from inside the Driver — the Driver remains opaque). This provides live observability into agent execution for debugging and monitoring during V1 development.
- **Cluster status** — Aggregate agent status for the Human dashboard

**Routing rules:**

| Artifact type | Source | Router action |
|---|---|---|
| `escalation` | Member Agent (has `parent_task_id`) | Translate to `resume_event` on Leader's parent task (Leader receives escalation in same Working Memory context). If Leader unavailable, notify Human that cluster is paused. |
| `escalation` | Leader Agent (no `parent_task_id`) | Forward to Human (via HTTP interface) |
| `confirmation_request` | Any agent (`risk_level: high` tool) | Forward to Human directly, bypassing Leader Agent |
| Human response to escalation | Human | Create `resume_event` with `authority=HUMAN`, publish to target agent |
| Human response to confirmation | Human | Create `resume_event` with `authority=HUMAN`, publish to requesting agent |
| All other Artifacts | Any agent | No intervention — normal Bus delivery |

**Key design decisions:**
- **ClusterRouter is the single routing authority for escalations.** All `escalation` Artifacts are published to the Bus and intercepted by ClusterRouter. Router uses `parent_task_id` to decide the route: if present, translate to `resume_event` on the parent task (keeping Leader's Working Memory context intact); if absent (Leader Agent), forward to Human. This avoids the deadlock that would occur if escalations were delivered as separate tasks while Leader is suspended in `wait_for_tasks`.
- **Leader Agent does not know about Router.** When Leader Agent needs Human help, its Driver returns ESCALATED → TaskRunner publishes an `escalation` Artifact to the Bus → Router picks it up and forwards to Human. Leader Agent only knows it published an escalation and eventually received a `resume_event`.
- **Confirmation results flow naturally.** When Human approves/rejects a `risk_level: high` tool, the `resume_event` goes to the requesting Member Agent. The sub-task result eventually reaches Leader Agent through the `parent_task_id` → `resume_event` mechanism — Leader Agent learns the outcome without needing a separate notification.
- **Router connects via Bus interceptor.** ClusterRouter registers an interceptor on the Bus (predicate: artifact type is `escalation` or `confirmation_request`). This ensures all matching artifacts are automatically routed through the Router before normal delivery, without agents needing to address artifacts to the Router explicitly.

### Leader Agent (Leader Role)

Leader Agent is a standard HAL instance with the `product_manager` role. It additionally loads cluster management tools. Member Agents do NOT have access to these tools.

**Task dispatch and collection:**

- **assign_task** — Assign a task to a specific Member Agent. Publishes a `task_assignment` Artifact (with `parent_task_id` set to Leader's current task ID) and returns the assigned `task_id`.
  - `target`: Member Agent instance ID
  - `task`: Task description
  - `position`: Queue insertion position — `"next"` (first in queue, default), `"last"` (end of queue), or `int` (specific position)
  - `suspend_current`: Whether to suspend a task to free a slot for this task (default: False). If True, Leader must specify `suspend_task_id` to indicate which task to preempt (for single-task agents with only one running task, this can be omitted). If the Driver supports `cancellation_token`, suspension takes effect between steps; otherwise, the new task starts after the current Driver call completes naturally (soft preemption).
  - `suspend_task_id`: The task ID to preempt (required when `suspend_current=True` and the target agent has multiple occupied slots). Leader uses `query_agent_tasks` to identify the target.
  - Returns `task_id` immediately (non-blocking). Leader uses `wait_for_tasks` to await results.
- **wait_for_tasks** — Wait for sub-task state changes. Reuses the standard PENDING → SUSPENDED tool protocol:
  1. Registers a watcher in the framework for the specified task IDs
  2. Returns PENDING with a `resume_event_id`
  3. Driver receives PENDING → stops internal loop → returns SUSPENDED
  4. When any watched sub-task changes state (completion, failure via AgentService `resume_event`; escalation via ClusterRouter `resume_event`), the framework delivers the event data to Leader's task
  5. Leader resumes with Working Memory preserved, receives event data (sub-task result, escalation content, etc.)
  - `task_ids`: List of task IDs to watch
  - Leader's LLM decides next steps — resolve escalation, dispatch follow-up tasks, call `wait_for_tasks` again for remaining tasks, or report to Human.

**Queue management:**

- **query_cluster_status** — Query all Member Agents' current capacity. Returns each agent's capacity snapshot (running/suspended/queued/preempted counts, available slots, max concurrency) and running task descriptions. Leader's LLM interprets capacity data and makes scheduling decisions autonomously.
- **query_agent_tasks** — Query a specific Member Agent's current task queue. Returns the running task(s) and the ordered list of queued tasks with their task IDs and descriptions.
- **reorder_agent_tasks** — Reorder a Member Agent's queued tasks. Leader provides the complete ordered list of task IDs; AgentService replaces its queue order accordingly. Only affects queued tasks — running tasks are not affected. Leader can combine this with `assign_task(suspend_current=True)` to fully control execution order.

**Escalation handling:**

- **resolve_escalation** — Respond to a Member Agent's escalation. Publishes a `resume_event` to the escalated Member Agent, allowing it to resume execution with the provided resolution.
  - `target_task_id`: The Member Agent's escalated task ID (from the received `escalation` event data in `resume_event`)
  - `resolution`: Resolution content (guidance, decision, or instructions for the Member Agent)

**Communication:**

- **collect_opinions** — Ask multiple Member Agents a question concurrently. Leader's LLM decides the queue position for each target (e.g., `position="next"` for urgent queries, `position="last"` for non-urgent). Includes a `timeout` parameter (passed to `wait_for_reply`) — agents that do not respond in time are marked as unavailable in the combined result, with the reason.
- **broadcast_decision** — Broadcast a `decision` artifact to specified Member Agents. Fire-and-forget (no wait for reply).

### Leader Agent Responsibilities

| Responsibility | Description |
|----------------|-------------|
| Task Decomposition | Break high-level goals into Member Agent-assignable sub-tasks, analyze dependencies |
| Scheduling & Ordering | Dispatch tasks to members with explicit queue positioning. For independent sub-tasks, dispatch in parallel (multiple `assign_task` calls, then `wait_for_tasks`). For dependent sub-tasks, dispatch in dependency order across suspend/resume cycles. Dynamically reorder member queues as priorities change. |
| Progress Aggregation | Query member status and task queues, synthesize overall progress for Human |
| Conflict Resolution | Resolve disagreements between Member Agents when possible |
| Quality Gate | Review Member Agent deliverables before reporting to Human |

### AgentService — Deployable Unit

Each agent instance runs as an independent async service, communicating through HAL Bus. The AgentService wraps an Agent, maintains an **ordered task queue**, and manages task lifecycle. **Leader Agent is the primary authority on task ordering** — AgentService executes tasks in the order given, without autonomous reordering. The one exception is preempted task auto-resume: when a slot becomes available, preempted tasks are automatically resumed ahead of queued tasks, with Leader notified via `task_status_update`.

**Core state:**

```
AgentService:
  task_queue: list[Task]                # Pending tasks, ordered by Leader-specified position
  running_tasks: list[TaskRunner]       # Currently executing (Driver active)
  suspended_tasks: list[Task]           # Suspended for external event (occupying slots, no active Driver)
  preempted_task_ids: list[str]         # Tasks preempted by Leader (persisted, NOT occupying slots)
                                        # This is an index only — full task state lives in checkpoint store
  max_concurrent: int                   # From role config (concurrency.max_concurrent_tasks, default: 1)
                                        # Intrinsic to the agent's role — NOT adjustable by Leader Agent at runtime

  # Queryable capacity (exposed via query_cluster_status / query_agent_tasks)
  capacity:
    max_concurrent: int
    running: int                        # len(running_tasks)
    suspended: int                      # len(suspended_tasks) — occupying slots but idle
    queued: int                         # len(task_queue)
    preempted: int                      # len(preempted_task_ids) — not occupying slots
    available_slots: int                # max_concurrent - running - suspended
```

**Slot occupancy rule:**

A slot is occupied by any task that is assigned to this agent, regardless of execution state. This includes actively running tasks (Driver executing) AND suspended tasks (waiting for external `resume_event`). Only COMPLETED, FAILED, or preempted tasks (moved to `preempted_task_ids`) release their slot. This means:

- An agent with `max_concurrent=1` and one SUSPENDED task has zero available slots — new tasks must either queue or preempt.
- An agent with `max_concurrent=3`, two running tasks and one SUSPENDED task has zero available slots.
- `available_slots = max_concurrent - len(running_tasks) - len(suspended-for-external-event tasks)`

This rule is uniform across single-task and parallel-capable agents.

**Artifact handling:**

- **`task_assignment`** → Insert into task queue at the specified position. If a slot is available, start immediately. If `suspend_current=True` and no slot available, preempt a current task (see preemption below) and start the new task. Otherwise, wait in queue. When completed or failed, AgentService publishes a `resume_event` on the parent task (via `parent_task_id`) delivering the result to Leader's Working Memory context. When escalated, TaskRunner publishes an `escalation` Artifact to Bus — ClusterRouter handles routing (see Escalation Routing).
- **`resume_event`** → If the target task is currently occupying a slot (suspended for external event), resume its TaskRunner from checkpoint with the event data. If the target task has been preempted (in `preempted_task_ids`), store the event data in its checkpoint and mark it as ready-to-resume; it will be resumed when a slot becomes available. If the resumed task completes, publish the result via `parent_task_id`.
- **`status_query`** → Return current capacity snapshot and ordered task queue (no task execution involved).
- **`reorder`** → Replace the task queue order with the Leader-specified order. Running tasks unaffected.

**Preemption (Leader-directed):**

Preemption is NOT autonomous — it only occurs when Leader explicitly requests it via `assign_task(suspend_current=True)`:

1. AgentService preempts the task specified by `suspend_task_id` (or the sole running task for single-task agents). The target can be an actively running task or a suspended-for-external-event task. Suspended tasks are preempted directly (no cancellation needed since no Driver is running — just move to `preempted_task_ids`).
2. If the selected task has an active Driver and the Driver supports `cancellation_token`, set the token. The Driver checks it between steps and returns early. AgentService saves checkpoint with SUSPENDED status.
3. If the selected task has an active Driver but does NOT support `cancellation_token` (soft preemption), the new task is queued at the front. When the current Driver call completes naturally, AgentService saves its result, then starts the new task before resuming queue processing. The `suspend_current` intent is fulfilled with a delay.
4. AgentService records the preempted task ID in `preempted_task_ids` (persisted), freeing the slot. When a slot becomes available, AgentService selects the next task to resume using this priority: (a) preempted tasks that have received a `resume_event` while preempted (ready-to-resume, external dependency satisfied — highest priority), (b) other preempted tasks in preemption order (first preempted, first resumed), (c) queued tasks from the task queue. If a preempted task has stored `resume_event` data, that data is injected on resume.
5. AgentService returns preemption confirmation to the Leader as part of the `assign_task` response. No separate `task_status_update` artifact is published for the preemption itself — Leader initiated it and already has full context.
6. **Auto-resume notification:** When a preempted task is automatically resumed (slot became available), AgentService publishes a `task_status_update` (type: resumed) to the parent task via `parent_task_id`, delivered as a `resume_event` to Leader. This ensures Leader is aware of all task state changes, even those initiated by the framework. Leader can re-preempt or reorder if the auto-resumed task conflicts with current priorities.

**Known limitation (V1):** No preemption depth limit. Cascading preemptions (multiple `suspend_current=True` in succession) cause repeated checkpoint/restore cycles. This is acceptable because Leader Agent has full visibility and control — it should not issue excessive preemptions. If this occurs, the root cause is Leader's scheduling strategy, not a framework gap.

**V1 persistence note:** `preempted_task_ids` is not separately persisted in V1 (single-process mode). On process restart, AgentService rebuilds the list from checkpoint store by scanning for checkpoints with SUSPENDED status belonging to this agent. This is sufficient for V1's single-process topology where process restart means full state loss anyway. V2+ distributed backends require explicit persistence of the preempted task list alongside checkpoints.

**Parallel task conflict awareness:**

For parallel-capable agents running multiple tasks concurrently, each new TaskRunner's context includes a summary of currently running tasks (task description and status), injected into Working Memory as a `[CONCURRENT TASKS]` message before the task description. The agent's LLM can observe potential conflicts (e.g., two tasks modifying the same module) and decide to merge its approach — for example, coordinating file edits or combining related changes into a single commit. This is the agent's own AI capability, not a framework-enforced mechanism. If the LLM does not handle a conflict, the underlying system (OS file locks, git merge conflicts) will report errors, and the Driver receives tool failures prompting re-planning.

Error isolation: each task is handled independently. One task's failure does not affect other running tasks in the same AgentService.

### Escalation Routing

Human is contacted through two paths: escalation (agent requests help) and high-risk tool confirmation (agent requests approval). All `escalation` and `confirmation_request` Artifacts are published to the Bus and intercepted by ClusterRouter. ClusterRouter routes based on context: Member Agent escalations (with `parent_task_id`) are translated to `resume_event`s on Leader's parent task; Leader Agent escalations (no `parent_task_id`) and all `confirmation_request`s are forwarded to Human. In all cases, the agent saves checkpoint and waits for a response:

```
Member Agent's Driver returns ESCALATED
  → TaskRunner saves checkpoint (sets suspended_at timestamp)
  → Publishes escalation Artifact to Bus (carries parent_task_id)
  → ClusterRouter intercepts, sees parent_task_id:
    → Translates to resume_event on Leader's parent task (Leader's Working Memory preserved)
    → Leader Agent's LLM sees escalation content in same conversation context
      → Leader Agent can resolve? → Calls resolve_escalation(target_task_id, resolution) → Member Agent resumes (Human unaware)
      → Leader Agent cannot resolve? → Leader Agent's Driver returns ESCALATED
        → Publishes escalation Artifact to Bus
        → ClusterRouter routes to Human (via HTTP interface)
          → Human responds
          → ClusterRouter creates resume_event (authority=HUMAN)
          → Leader Agent resumes, calls resolve_escalation to relay resolution to Member Agent
    [V1: Leader Agent unavailable]
    → ClusterRouter notifies Human "Leader Agent unavailable, cluster paused"
    → Cluster stops until Human intervention (restart)
    [V2+: Degradation path — see V2 roadmap]
```

**High-risk tool confirmation (separate path):**

```
Member Agent calls a risk_level: high tool (e.g., deploy)
  → SandboxedTool returns PENDING (confirmation_request)
  → Driver returns SUSPENDED (with suspend_context.confirmation_request)
  → TaskRunner publishes confirmation_request Artifact to Bus
  → ClusterRouter routes directly to Human (bypasses Leader Agent)
    → Human approves/rejects
    → ClusterRouter creates resume_event (authority=HUMAN)
    → Member Agent resumes (executes or aborts the tool)
    → Task eventually completes → result reaches Leader Agent via parent_task_id → resume_event
```

BUDGET_EXHAUSTED follows the escalation path — Leader Agent or Human decides whether to allocate more budget.

External system rejections (git permission denied, build failure) are NOT escalations. They are tool failures that the Driver's LLM handles via normal re-planning.

### External Event Sources for `resume_event` Artifacts

| Source | Example | Mechanism |
|--------|---------|-----------|
| Other HAL instance | Code review completed, test passed | Direct Artifact via HAL Bus |
| External system webhook | GitLab MR approved, CI pipeline passed | **Webhook Adapter**: standalone HTTP service → `resume_event` Artifact |
| Human | Manually completed something outside HAL | ClusterRouter HTTP interface → `resume_event` |

The Webhook Adapter is a standalone service, not part of HAL core. Its dependencies on HAL are minimal and well-defined: (1) the `resume_event` Artifact schema (for constructing valid Artifacts), (2) the Bus connection configuration (e.g., Redis address for RedisBus), and (3) the target agent's instance ID (to address the Artifact). The adapter has no dependency on any other HAL component — it publishes directly to the Bus like any other producer.

---

## Deployment

### Current: Local Mode (single process)

All Agents + ClusterRouter run as asyncio coroutines in one Python process. Communication via InMemoryBus. No external dependencies.

```
Single Python Process
┌─────────────────────────────────────────────────────────────┐
│  asyncio.gather(                                            │
│      cluster_router.start(),                                │
│      leader_agent_service.start(),                              │
│      ios_dev_service.start(),                               │
│      android_dev_service.start(),                           │
│      qa_service.start(),                                    │
│  )                                                          │
│  ┌──────────┐                                               │
│  │ Cluster  │ ← HTTP/WS (Human interface)                   │
│  │ Router   │                                               │
│  └────┬─────┘                                               │
│       │                                                     │
│  ┌──────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐          │
│  │  Leader  │ │ iOS Dev │ │ And Dev │ │   QA    │          │
│  │  Agent   │ │  Agent  │ │  Agent  │ │  Agent  │          │
│  │  memory  │ │ memory  │ │ memory  │ │ memory  │          │
│  │  prompt  │ │ prompt  │ │ prompt  │ │ prompt  │          │
│  │  tools   │ │ tools   │ │ tools   │ │ tools   │          │
│  │  driver  │ │ driver  │ │ driver  │ │ driver  │          │
│  └────┬─────┘ └────┬────┘ └────┬────┘ └────┬────┘          │
│       └──────┬─────┴──────┬────┘           │                │
│              ▼            ▼                ▼                │
│         ┌──────────────────────────────────┐                │
│         │   InMemoryBus (async, in-process)│                │
│         └──────────────────────────────────┘                │
└─────────────────────────────────────────────────────────────┘
```

Context isolation guaranteed at the object level. Each Agent instance holds its own Working Memory, tools, system prompt, and Driver instance. Agents are IO-bound (waiting for LLM API responses), so a single event loop can efficiently serve a cluster of 5-10 agents.

**Async requirement:** All Driver implementations MUST be cooperative async — no blocking calls that would starve other agents on the shared event loop. Leader tools like `collect_opinions` use async `wait_for_reply` which yields to the event loop, allowing Member Agents to process their tasks concurrently. `wait_for_tasks` causes Leader's Driver to return SUSPENDED, freeing the event loop entirely until sub-task state changes trigger a `resume_event`.

**Graceful shutdown:** On SIGTERM, the cluster performs an orderly shutdown: (1) all AgentServices stop accepting new tasks from the Bus, (2) wait for all active Driver calls to return naturally and checkpoint all in-progress tasks. A configurable `shutdown_timeout` (default: 10 seconds) limits the wait. Behavior after timeout is controlled by `shutdown_policy`:
- `prompt_human` (default) — Human is prompted to confirm whether to force exit.
- `checkpoint_and_exit` — Automatically checkpoint all in-progress tasks and exit without prompting. Suitable for non-interactive deployments (CI/CD, cron-triggered runs) where no Human is available to respond.

In both cases, tasks that did not complete fall back to their last checkpoint, potentially losing the current Driver round's progress.

### Architecture Extensibility (not implemented, design-ready)

HAL Bus is a pluggable Protocol. All agent communication (AgentService, Leader tools, escalation routing) goes through the Bus interface, never directly between agents. This creates two independent dimensions for deployment flexibility:

**Dimension 1: Bus backend** — determines the communication boundary
- InMemoryBus → agents within the same process only
- Network Bus (e.g., RedisBus) → agents across processes and machines

**Dimension 2: Process model** — determines how agents are hosted
- Coroutine → multiple agents share one process (lightweight, zero IPC overhead)
- Process → each agent is an independent OS process (fault isolation, independent restart)

These two dimensions are orthogonal. This enables three deployment topologies beyond the current single-process mode:

**Multi-process (single machine):** Each agent as a separate process, communicating via SQLiteBus — a shared SQLite WAL file on local disk. Zero external dependencies, cross-platform (Windows/macOS/Linux). Provides process-level fault isolation. Bus file stored in project directory (e.g., `{project}/.hal/bus.db`).

**Distributed (multiple machines):** Agents on different machines, communicating via RedisBus. Requires a shared Redis server. Provides horizontal scaling.

**Hybrid:** Some agents as coroutines in one process, others as separate processes or on remote machines, all sharing the same network Bus. ClusterRouter always runs alongside the Leader Agent (same machine). For example, Router + Leader Agent + Dev agents on a developer's machine (one process), QA agent on a CI server (separate process) — connected through a shared Redis:

```
Developer Machine (one process)              CI Server (separate process)
┌──────────────────────────────────┐        ┌──────────────┐
│ ClusterRouter + Leader Agent         │        │ QA Agent     │
│ + iOS + Android + HarmonyOS      │        │              │
│ (coroutines)                     │        │              │
└───────────┬──────────────────────┘        └──────┬───────┘
            │                                      │
            ▼                                      ▼
     ┌─────────────────────────────────────────────────┐
     │          Redis (shared HAL Bus)                 │
     └─────────────────────────────────────────────────┘
```

Note: SQLiteBus only works for agents on the same machine (SQLite does not support network file systems). Cross-machine scenarios require RedisBus.

**Topology → Bus mapping:**

| Topology | Bus backend | External dependency |
|----------|-------------|-------------------|
| Single process | InMemoryBus | None |
| Multi-process, same machine | SQLiteBus | None |
| Cross-machine / Hybrid | RedisBus | Redis |

All topologies require only:
- The appropriate Bus backend implementation
- A `bus` section in cluster config (omit for InMemoryBus default)
- No changes to: AgentDriver, TaskRunner, Leader tools, escalation routing, Artifacts, Role profiles, tools, memory, checkpoints

Agents are unaware of the topology — they publish and subscribe through the Bus Protocol. A coroutine in a shared process and a process on a remote machine behave identically from the framework's perspective.

### Configuration

Leader Agent is a full HAL instance — same as any Member Agent (own role, tools, Driver). The only differences: it additionally loads cluster management tools (assign_task, wait_for_tasks, query_cluster_status, query_agent_tasks, reorder_agent_tasks, resolve_escalation, collect_opinions, broadcast_decision). Leader Agent must be explicitly designated in the cluster config. Each cluster has exactly one Leader Agent — zero or multiple is a config validation error.

ClusterRouter is configured separately — it is infrastructure, not a HAL instance.

```yaml
cluster:
  name: "mobile-sdk-team"
  max_cost_per_task: 50.0           # Hard cost ceiling per task in USD (default: 50.0, failsafe)
  shutdown_policy: prompt_human     # prompt_human (default) or checkpoint_and_exit
  bus:
    max_inbox_size: 100             # Per-agent inbox capacity (default: 100)
  router:
    human_interface: http         # HTTP/WebSocket for Human interaction
    listen: "0.0.0.0:8080"
  leader_agent:                       # The designated leader instance
    role: product_manager
    instance: "pm-01"
  member_agents:
    - role: ios_developer
      instance: "ios-dev-01"
    - role: android_developer
      instance: "android-dev-01"
    - role: harmonyos_developer
      instance: "harmony-dev-01"
```

---

## Memory System

Three-tier architecture. Working Memory is owned by HAL (passed to/from the Driver). Episodic and Shared Memory provide persistent knowledge across tasks.

### Working Memory (per-task, ephemeral)

Current task context — the conversation history passed to the Driver. HAL owns this state: it appends HAL-originated messages, replaces the full history with the Driver's updated version after execution, and snapshots/restores for checkpointing. Discarded when the task completes.

**Context window management is the Driver's responsibility.** When conversation history approaches the LLM's context limit, the Driver handles truncation, summarization, or compression internally. HAL does not monitor token usage or enforce context window limits — messages are opaque to HAL. Checkpoint stores the Driver's returned messages as-is (which may already be compressed by the Driver). This means a checkpoint reflects the Driver's view of the conversation, not the full uncompressed history.

### Episodic Memory (per-instance, persistent)

Lessons learned from past tasks, accumulated by each agent instance. Each entry carries: type, content, source task, trigger (what happened), confidence score, validation count, last-used timestamp, and tags for retrieval.

Example:

```yaml
type: lesson_learned
source_task: "feat-login-sdk"
trigger: build_failure
content: |
  CocoaPods 1.15 is incompatible with Xcode 16.
  Always pin to 1.14 in Podfile.
confidence: 0.88
times_validated: 3
tags: [cocoapods, xcode, build]
```

### Shared Memory (cluster-wide, persistent)

Knowledge shared across all instances in a cluster: project architecture decisions, cross-platform API contracts, team coding conventions. Only Leader Agent can write (see Shared Memory write policy in Memory Optimizations).

Each entry carries:

```yaml
content: "iOS 和 Android 的登录 SDK 必须使用相同的 token 刷新策略..."
tags: [login, token, cross-platform]
confidence: 0.9               # Leader-written entries start at 0.9 (max cap)
created_by: "pm-01"           # Writer (Leader instance ID)
created_at: "2026-04-28T..."
```

Shared Memory entries are written by Leader Agent with initial confidence of 0.9 (the cap). Like Episodic Memory, they are subject to confidence decay and eviction via the same score formula (confidence × recency). This ensures stale cross-team knowledge is eventually evicted even if it was authoritative when written. Eviction follows the Hard cap + LRU policy (`max_shared`, default: 500).

### Episodic Memory Updates — Driver-Reported Lessons

In the Driver model, HAL cannot observe individual steps during execution, and messages are opaque to HAL. Episodic memory updates are the **Driver's responsibility**.

DriverResult carries an optional lessons field — a list of structured lesson entries that the Driver extracts during execution. Each lesson includes: content, trigger (what happened), confidence, tags, and **scope** (`episodic` or `shared`). HAL routes lessons based on scope:
- `scope: episodic` → stored in the agent's Episodic Memory (default for all agents)
- `scope: shared` → stored in cluster-wide Shared Memory (only effective for Leader Agent; if a Member Agent reports `scope: shared`, it is downgraded to `episodic`)

The `auto` / `human_confirm` write policy (configured in cluster config) applies at the routing point: when a `scope: shared` lesson arrives from Leader Agent, the framework either writes it directly (`auto`) or routes it to Human for confirmation via ClusterRouter (`human_confirm`).

**Initial confidence cap:** Driver-reported lessons and status-based inferred lessons are capped at 0.7 confidence on initial storage, regardless of the Driver's self-reported confidence value. Confidence above 0.7 can only be achieved through cross-task validation (the same lesson confirmed in a different task context). This prevents a buggy or hallucinating Driver from injecting high-confidence incorrect knowledge that resists decay. Exception: Leader Agent's `scope: shared` lessons are stored with initial confidence 0.9 (the cap), since they represent deliberate cross-team knowledge decisions by the coordination role.

Additionally, HAL performs lightweight status-based inference from DriverResult's structured fields (not from messages):
- FAILED status + summary → generate a negative lesson from the summary text (initial confidence 0.7, same cap as Driver-reported lessons)
- BUDGET_EXHAUSTED → record task complexity as a planning hint (initial confidence 0.7)

Drivers that do not populate the lessons field simply produce no episodic updates beyond the status-based inference. This is acceptable — memory extraction is a progressive enhancement, not a hard requirement.

### Memory Retrieval

Memories are not injected wholesale. Retrieval supports two modes depending on configuration:

**Tag-based matching (V1, zero dependencies):** Memories are matched by tag overlap with the current task's keywords. Simple, no external services required. This is the only retrieval mode implemented in V1.

**Vector-based semantic search (V2+, requires embedding config):** Each memory gets an embedding vector. Task description used to match top-K most relevant (e.g., 10). Pure math — millisecond latency, no token cost. Requires an embedding source configured separately from the AgentDriver (e.g., an API endpoint for a lightweight embedding model). HAL calls the embedding API directly for memory indexing — this is the only LLM-adjacent call HAL makes, independent of the Driver. This is a V2+ progressive enhancement.

In both modes, **summary compression** reduces injection cost: each memory has a full version (for storage) and a summary version (for injection into system prompt). Default injection uses summaries.

### Memory Optimizations

1. **Consolidation** — New memories matched against existing before storage. Similar entries merged (boost confidence, keep more detailed version). V1 uses tag-based matching for similarity detection; V2+ can use vector similarity when embedding is enabled.
2. **Hard cap + LRU eviction** — Episodic max 200 entries (default), Shared max 500 (default). Configurable per role in role profile (`memory.max_episodic`, `memory.max_shared`). Score = confidence × recency. Lowest-scoring permanently deleted (no archive). Applies uniformly to both Episodic and Shared Memory.
3. **Tiered injection** — Retrieve at task start only, inject into system prompt (~500 tokens). No retrieval during normal execution steps. **Known limitation:** For long-running single Driver calls, memories produced by other agents during execution are not visible until the next Driver call (e.g., after suspend/resume). This is an acceptable V1 trade-off — mid-execution memory refresh would require Driver cooperation and adds complexity with marginal benefit for typical task durations.
4. **Shared Memory write policy** — Member Agents cannot write directly. All cross-role knowledge flows through Leader Agent via the `scope: shared` field in DriverResult lessons. Configurable: `auto` (Leader Agent's `scope: shared` lessons written directly) or `human_confirm` (Human reviews via ClusterRouter HTTP interface before write).
5. **Deferred batch embedding (V2+)** — New memories stored as text immediately, embedded in batches. Only relevant when vector-based retrieval is enabled.

### Memory Persistence

Pluggable backends, same pattern as Checkpoint: InMemoryStore (testing), SQLiteStore (production). Episodic Memory is per-instance (each agent's store is independent). Shared Memory is cluster-wide (single shared store, write-access restricted to Leader Agent).

### Memory Hygiene

- **Confidence decay** — Memories lose confidence over time when not used. Decay is computed lazily at retrieval time (no background scan): `effective_confidence = confidence × 0.99 ^ days_since_last_used`. A memory unused for 60 days decays to ~55% of its stored confidence. A memory is considered "used" when it is retrieved and injected into the system prompt — its `last_used` timestamp is updated at that point. Decay parameters (`decay_factor`, default: 0.99) are configurable per role.
- **Confidence cap** — Maximum confidence is capped at 0.9. Even through repeated cross-task validation, no memory can reach 1.0. Combined with decay, this ensures incorrect memories that are no longer validated will eventually be evicted.
- **Validation reinforcement** — Re-confirmed lessons gain confidence (up to the 0.9 cap).
- **Contradiction resolution** — New lesson conflicts with old? Keep higher confidence.

---

## Technology Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Agent execution | AgentDriver protocol | Delegate to battle-tested SDKs; HAL focuses on coordination |
| Implementation | From scratch | Full control, no framework coupling, lightweight |
| Language | Python 3.12+ | Mature AI/Agent ecosystem |
| Async framework | asyncio | Python stdlib, mature ecosystem |
| Message queue | InMemoryBus (pluggable, Redis Streams ready) | Start simple; Bus Protocol enables future distributed backends without code changes |
| HTTP framework | aiohttp or FastAPI | ClusterRouter's Human interface |
| Persistence | Pluggable (SQLite default) | Local dev simplicity, production flexibility |
| Vector storage | V2+. SQLite + vector extension when enabled | V1 uses tag-based matching only (zero dependencies); vector search planned as V2+ progressive enhancement |
| Design inspiration | LangGraph concepts | State graph, checkpoint, HITL patterns — as design ideas, not dependency |

---

## Estimated Scope

### V1 Scope

| Module | LOC (est.) | Notes |
|--------|-----------|-------|
| Core Kernel (Config, AsyncEventBus, Logger, Plugins, Errors, AgentDriver protocol, HAL Bus + Artifacts + Authority) | 900-1,200 | Idempotency simplified for single-process |
| Tool Arsenal (Protocol, Registry, Loader, SandboxedTool with visibility control + risk_level confirmation) | 500-700 | No path-level sandbox |
| Identity & Role (Profile, Boundaries, Knowledge, Concurrency config) | 300-500 | |
| Task Runner (TaskRunner, Working Memory, Checkpoint, Safeguards, Escalation, Suspend TTL, Cancellation) | 500-700 | |
| Memory System (Episodic, Shared, Tag-based Retrieval, Hygiene) | 300-400 | No vector search |
| Cluster (Config, Leader Agent tools, AgentService + Ordered Task Queue + Preemption, ClusterRouter) | 800-1,000 | No degradation mode |
| ClaudeCodeDriver (V1 reference Driver implementation) | 200-400 | |
| MockDriver (for testing) | 100-200 | |
| Streaming Log (WebSocket event stream for live observability) | 100-150 | |
| Configuration & CLI | 300-500 | |
| **Total (excluding tests)** | **~4,000-5,750** | |
| Tests (1:1 ratio) | ~4,000-5,750 | |
| **Grand total** | **~8,000-11,500** | |

### V2+ Roadmap

| Feature | Description |
|---------|-------------|
| Vector-based memory retrieval | Embedding API integration, semantic search, deferred batch embedding |
| Path-level sandbox | Filesystem path restrictions in SandboxedTool, combined with OS-level isolation (Docker/chroot/Seatbelt/Landlock) |
| Leader Agent degradation mode | Direct Human forwarding when Leader unavailable, `degradation_log`, `catch_up` Artifact |
| Distributed deployment | SQLiteBus (multi-process), RedisBus (cross-machine), Webhook Adapter |
