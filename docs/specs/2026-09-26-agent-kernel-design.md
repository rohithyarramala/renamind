# Agent Kernel — Design Spec

**Date:** 2026-09-26
**Subsystem:** Phase 1 (Foundation) — [ROADMAP.md](../../ROADMAP.md)
**Crate:** `crates/rena-kernel`
**Status:** Approved — implementation in progress

## Purpose

The Agent Kernel is a generic orchestration primitive: a `Bot` trait plus a
runtime that registers bots, routes tasks to them by declared capability,
supervises them through failure, and lets them hand off work to each other
or pause for human approval. It ships with zero or one trivial test bot,
not a suite of real specialized bots ("researcher," "coder," etc.) — those
are built on top of it later, each its own effort. In the OS-kernel sense:
it schedules arbitrary work, it isn't the work itself.

## Non-goals (explicit v1 boundaries)

- **No persistence beyond process lifetime.** In-memory only. No
  speculative trait hook for the Memory Store either — that subsystem
  doesn't have a spec yet, so designing against it now would be guessing.
- **No real LLM calls.** `ModelClient` is mocked; real OpenRouter wiring is
  the LLM Gateway subsystem's own spec.
- **No real human-notification channel.** `ApprovalChannel` ships as a
  programmable stub (always-approve / always-reject / timeout) for tests
  only. Slack/dashboard/CLI integration is future work.
- **No RBAC or permission checks** on task submission — Phase 2.
- **No multi-step workflows/DAGs.** The kernel does single-hop delegation
  only; multi-step visual/code workflows are the Automation Engine's job
  (Phase 4). This boundary is deliberate so the kernel doesn't grow into a
  workflow engine by accident.
- **No distributed/multi-node execution.** Single process for v1.
- **No hot-loadable/plugin bots.** Bots are registered in code at startup.

## Concurrency architecture

Three options were considered:

- **A — Hand-rolled actor-lite on raw Tokio (chosen).** Each bot is a
  `tokio::spawn`'d task with an `mpsc` inbox; the kernel holds sender
  handles in a capability-indexed registry. Zero extra dependencies for
  the trust-critical core of the system.
- **B — Adopt an existing actor framework** (`ractor`, `actix`, `coerce`).
  Gives supervision trees for free but adds a dependency (and its
  opinions) to the one crate that most needs a small, auditable surface.
  Rejected for v1.
- **C — Multi-process kernel** (bots as OS subprocesses over IPC).
  Strongest isolation, relevant once real multi-tenant sandboxing matters
  — that's a Phase 2+ concern tied to RBAC/Guardrails, not v1. Rejected as
  premature.

## Core types (illustrative — refined during implementation)

```rust
pub type Capability = String;

pub struct Task {
    pub id: TaskId,
    pub capability: Capability,
    pub payload: serde_json::Value,
    pub delegation_depth: u8,   // guards against infinite handoff loops
}

pub enum BotOutcome {
    Done(TaskOutput),
    Delegate { to_capability: Capability, task: Task },
    NeedsApproval { request: ApprovalRequest, resume: Task },
    Failed(BotError),
}

pub struct TaskOutput {
    pub result: serde_json::Value,
    pub confidence: f32,   // real scoring logic lands in a later cross-cutting spec
}

#[async_trait]
pub trait Bot: Send + Sync {
    fn capabilities(&self) -> &[Capability];
    async fn handle(&self, task: Task, models: &dyn ModelClient) -> BotOutcome;
}

#[async_trait]
pub trait ModelClient: Send + Sync {   // boundary to the not-yet-built LLM Gateway
    async fn complete(&self, prompt: &str) -> Result<String, ModelError>;
}

pub struct ApprovalRequest {
    pub summary: String,
    pub payload: serde_json::Value,
    pub risk_note: Option<String>,
}

pub enum ApprovalDecision {
    Approved,
    Rejected { reason: Option<String> },
}

#[async_trait]
pub trait ApprovalChannel: Send + Sync {
    async fn request(&self, req: ApprovalRequest) -> ApprovalDecision;
}

pub struct Kernel {
    registry: HashMap<Capability, Vec<BotHandle>>,
    models: Arc<dyn ModelClient>,
    approvals: Arc<dyn ApprovalChannel>,
    max_delegation_depth: u8,
    approval_timeout: Duration,
}
```

## Routing

`submit_task` looks up `registry[task.capability]`. v1 rule: first
available bot, round-robin when more than one bot shares a capability.
Simple and testable; can grow smarter (load-aware, priority-aware) later
without breaking the public API.

## Fault handling / supervision

Each bot's task loop runs inside `tokio::spawn`, so a panic there fails
only that `JoinHandle`, never the process. The kernel's supervisor catches
the resulting `JoinError`, logs it, and respawns that bot with bounded
exponential backoff up to a retry cap. Past the cap, that capability is
marked degraded and further tasks routed to it return `KernelError`
immediately instead of hanging.

## Delegation

A `BotOutcome::Delegate` re-enters routing with `delegation_depth + 1`.
Exceeding `max_delegation_depth` is a hard error
(`KernelError::DelegationLimitExceeded`) — this is the guard against
infinite handoff loops between bots.

## Human-in-the-loop

A `BotOutcome::NeedsApproval` carries the `resume` task to continue with
once approved. The kernel calls `ApprovalChannel::request` with a bounded
timeout (`approval_timeout`, configurable per `Kernel` instance — default
24 hours, since solo and enterprise contexts will want very different
values here; tests set short timeouts explicitly). Behavior is **fail closed**: a
timeout is treated identically to `Rejected`, never as `Approved`. Only an
explicit `Approved` decision resumes the bot with `resume`; `Rejected`
(explicit or via timeout) surfaces as `KernelError::ApprovalDenied` with no
retry. This is a new cross-cutting concept the kernel introduces the hook
for — RBAC will later define *who* may approve, Observability will track
approval latency, but neither needs to exist for the kernel's contract to
be complete.

## Persistence

In-memory only. No task history survives a restart in v1 — a known,
explicit limitation, not an oversight.

## Testing plan

- Panicking mock bot → verify restart with backoff, then degradation past
  the retry cap.
- Self-delegating mock bot → verify the max-delegation-depth guard fires.
- Multiple bots registered under one capability → verify round-robin.
- Concurrent task submission → verify no cross-task interference.
- Mock bot returning `NeedsApproval` → verify `Approved` resumes,
  `Rejected` surfaces `KernelError`, and a timeout behaves identically to
  `Rejected` (fail closed).

## Crate layout

Cargo workspace at the repo root; this subsystem lives at
`crates/rena-kernel`. Later subsystems get their own crates the same way,
matching the boundaries already drawn in [ROADMAP.md](../../ROADMAP.md).
