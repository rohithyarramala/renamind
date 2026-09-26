# Agent Kernel Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `crates/rena-kernel`, a working, tested Rust crate implementing the Agent Kernel exactly as specified in [docs/specs/2026-09-26-agent-kernel-design.md](../specs/2026-09-26-agent-kernel-design.md): capability-routed bot orchestration, panic supervision with bounded backoff, single-hop delegation with a depth guard, and a fail-closed human-in-the-loop approval hook.

**Architecture:** A Cargo workspace with one crate. Bots are `Arc<dyn Bot>` registered under capability strings; the kernel spawns a fresh Tokio task per dispatch attempt so a panic fails only that task, never the process; retries happen by re-spawning with backoff. `ModelClient` and `ApprovalChannel` are traits the kernel depends on abstractly, with stub implementations for tests only — no real LLM or human-notification integration in this plan.

**Tech Stack:** Rust, Tokio (async runtime), `async-trait`, `serde`/`serde_json`, `thiserror`, `uuid`, `tracing`.

---

### Task 1: Workspace + core types

**Files:**
- Create: `Cargo.toml` (workspace root)
- Create: `crates/rena-kernel/Cargo.toml`
- Create: `crates/rena-kernel/src/lib.rs`
- Create: `crates/rena-kernel/src/types.rs`

- [ ] **Step 1: Create the workspace root `Cargo.toml`**

```toml
[workspace]
resolver = "2"
members = ["crates/rena-kernel"]
```

- [ ] **Step 2: Create `crates/rena-kernel/Cargo.toml`**

```toml
[package]
name = "rena-kernel"
version = "0.1.0"
edition = "2021"
description = "Agent Kernel: bot orchestration primitive for RenaMind"

[dependencies]
tokio = { version = "1", features = ["rt-multi-thread", "macros", "sync", "time"] }
async-trait = "0.1"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
thiserror = "1"
uuid = { version = "1", features = ["v4"] }
tracing = "0.1"
```

- [ ] **Step 3: Write the failing test for core types**

Create `crates/rena-kernel/src/types.rs` with the test at the bottom first:

```rust
// (types below the test are written in Step 5 — for now only the test
// module exists, so this fails to compile.)

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn task_new_starts_at_delegation_depth_zero() {
        let task = Task::new("echo", serde_json::json!({"hello": "world"}));
        assert_eq!(task.delegation_depth, 0);
        assert_eq!(task.capability, "echo");
    }

    #[test]
    fn task_ids_are_unique() {
        let a = Task::new("echo", serde_json::json!(null));
        let b = Task::new("echo", serde_json::json!(null));
        assert_ne!(a.id, b.id);
    }
}
```

- [ ] **Step 4: Wire `lib.rs` to declare the module, and run the test to verify it fails**

`crates/rena-kernel/src/lib.rs`:

```rust
//! Agent Kernel: a generic orchestration primitive for RenaMind bots.
//!
//! See `docs/specs/2026-09-26-agent-kernel-design.md` for the full design.

mod types;
```

Run: `cd crates/rena-kernel && cargo test task_new_starts_at_delegation_depth_zero`
Expected: FAIL to compile — `Task` is not defined.

- [ ] **Step 5: Implement the core types above the test module**

Prepend this to `crates/rena-kernel/src/types.rs` (above the `#[cfg(test)]` block):

```rust
use std::fmt;

use serde::{Deserialize, Serialize};
use uuid::Uuid;

pub type Capability = String;

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct TaskId(pub Uuid);

impl TaskId {
    pub fn new() -> Self {
        TaskId(Uuid::new_v4())
    }
}

impl Default for TaskId {
    fn default() -> Self {
        Self::new()
    }
}

impl fmt::Display for TaskId {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}", self.0)
    }
}

#[derive(Debug, Clone)]
pub struct Task {
    pub id: TaskId,
    pub capability: Capability,
    pub payload: serde_json::Value,
    pub delegation_depth: u8,
}

impl Task {
    pub fn new(capability: impl Into<Capability>, payload: serde_json::Value) -> Self {
        Task {
            id: TaskId::new(),
            capability: capability.into(),
            payload,
            delegation_depth: 0,
        }
    }
}

#[derive(Debug, Clone)]
pub struct TaskOutput {
    pub result: serde_json::Value,
    pub confidence: f32,
}

impl TaskOutput {
    pub fn new(result: serde_json::Value, confidence: f32) -> Self {
        TaskOutput { result, confidence }
    }
}

#[derive(Debug, Clone)]
pub struct ApprovalRequest {
    pub summary: String,
    pub payload: serde_json::Value,
    pub risk_note: Option<String>,
}

#[derive(Debug, Clone, PartialEq)]
pub enum ApprovalDecision {
    Approved,
    Rejected { reason: Option<String> },
}

#[derive(Debug)]
pub enum BotOutcome {
    Done(TaskOutput),
    Delegate { to_capability: Capability, task: Task },
    NeedsApproval { request: ApprovalRequest, resume: Task },
    Failed(BotError),
}

#[derive(Debug, thiserror::Error)]
#[error("{0}")]
pub struct BotError(pub String);

#[derive(Debug, thiserror::Error)]
pub enum ModelError {
    #[error("model call failed: {0}")]
    Failed(String),
}

#[derive(Debug, thiserror::Error)]
pub enum KernelError {
    #[error("no bot registered for capability '{0}'")]
    NoCapableBot(Capability),
    #[error("delegation limit exceeded for capability '{0}'")]
    DelegationLimitExceeded(Capability),
    #[error("approval denied: {0:?}")]
    ApprovalDenied(Option<String>),
    #[error("bot failed: {0}")]
    BotFailed(String),
    #[error("capability '{0}' is degraded after repeated failures")]
    Degraded(Capability),
}
```

- [ ] **Step 6: Run the tests to verify they pass**

Run: `cd crates/rena-kernel && cargo test types::tests`
Expected: PASS (2 tests)

- [ ] **Step 7: Commit**

```bash
git add Cargo.toml crates/rena-kernel/Cargo.toml crates/rena-kernel/src/lib.rs crates/rena-kernel/src/types.rs
git commit -m "feat(kernel): add workspace scaffold and core types"
```

---

### Task 2: Bot / ModelClient / ApprovalChannel traits + test doubles

**Files:**
- Create: `crates/rena-kernel/src/bot.rs`
- Create: `crates/rena-kernel/src/model_client.rs`
- Create: `crates/rena-kernel/src/approval.rs`
- Create: `crates/rena-kernel/src/test_support.rs`
- Modify: `crates/rena-kernel/src/lib.rs`

- [ ] **Step 1: Write the trait definitions**

`crates/rena-kernel/src/bot.rs`:

```rust
use async_trait::async_trait;

use crate::model_client::ModelClient;
use crate::types::{BotOutcome, Capability, Task};

#[async_trait]
pub trait Bot: Send + Sync {
    fn capabilities(&self) -> &[Capability];
    async fn handle(&self, task: Task, models: &dyn ModelClient) -> BotOutcome;
}
```

`crates/rena-kernel/src/model_client.rs`:

```rust
use async_trait::async_trait;

use crate::types::ModelError;

#[async_trait]
pub trait ModelClient: Send + Sync {
    async fn complete(&self, prompt: &str) -> Result<String, ModelError>;
}
```

`crates/rena-kernel/src/approval.rs`:

```rust
use async_trait::async_trait;

use crate::types::{ApprovalDecision, ApprovalRequest};

#[async_trait]
pub trait ApprovalChannel: Send + Sync {
    async fn request(&self, req: ApprovalRequest) -> ApprovalDecision;
}
```

- [ ] **Step 2: Write the test doubles used by every later test**

`crates/rena-kernel/src/test_support.rs`:

```rust
#![cfg(test)]

use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Arc;

use async_trait::async_trait;

use crate::approval::ApprovalChannel;
use crate::bot::Bot;
use crate::model_client::ModelClient;
use crate::types::{
    ApprovalDecision, ApprovalRequest, BotOutcome, Capability, ModelError, Task, TaskOutput,
};

pub struct MockModelClient;

#[async_trait]
impl ModelClient for MockModelClient {
    async fn complete(&self, prompt: &str) -> Result<String, ModelError> {
        Ok(format!("mock-response-to: {prompt}"))
    }
}

#[derive(Clone)]
pub enum StubApproval {
    Approve,
    Reject(Option<String>),
    Never,
}

pub struct StubApprovalChannel {
    pub behavior: StubApproval,
}

#[async_trait]
impl ApprovalChannel for StubApprovalChannel {
    async fn request(&self, _req: ApprovalRequest) -> ApprovalDecision {
        match &self.behavior {
            StubApproval::Approve => ApprovalDecision::Approved,
            StubApproval::Reject(reason) => ApprovalDecision::Rejected {
                reason: reason.clone(),
            },
            StubApproval::Never => {
                // The kernel's own timeout must fire in this case, not this mock.
                std::future::pending::<()>().await;
                unreachable!("pending future never resolves")
            }
        }
    }
}

pub enum TestBotBehavior {
    Succeed {
        result: serde_json::Value,
        confidence: f32,
    },
    Panic,
    DelegateTo(Capability),
    NeedsApproval {
        request: ApprovalRequest,
        resume_result: serde_json::Value,
    },
}

pub struct TestBot {
    pub name: String,
    pub capabilities: Vec<Capability>,
    pub behavior: TestBotBehavior,
    pub call_count: Arc<AtomicUsize>,
}

impl TestBot {
    pub fn new(name: &str, capability: &str, behavior: TestBotBehavior) -> Self {
        TestBot {
            name: name.to_string(),
            capabilities: vec![capability.to_string()],
            behavior,
            call_count: Arc::new(AtomicUsize::new(0)),
        }
    }
}

#[async_trait]
impl Bot for TestBot {
    fn capabilities(&self) -> &[Capability] {
        &self.capabilities
    }

    async fn handle(&self, task: Task, _models: &dyn ModelClient) -> BotOutcome {
        self.call_count.fetch_add(1, Ordering::SeqCst);
        match &self.behavior {
            TestBotBehavior::Succeed { result, confidence } => {
                BotOutcome::Done(TaskOutput::new(result.clone(), *confidence))
            }
            TestBotBehavior::Panic => panic!("TestBot '{}' panicking on purpose", self.name),
            TestBotBehavior::DelegateTo(cap) => BotOutcome::Delegate {
                to_capability: cap.clone(),
                task,
            },
            TestBotBehavior::NeedsApproval {
                request,
                resume_result,
            } => {
                let mut resume = task.clone();
                resume.payload = resume_result.clone();
                BotOutcome::NeedsApproval {
                    request: request.clone(),
                    resume,
                }
            }
        }
    }
}
```

- [ ] **Step 3: Wire the new modules into `lib.rs`**

Replace the contents of `crates/rena-kernel/src/lib.rs` with:

```rust
//! Agent Kernel: a generic orchestration primitive for RenaMind bots.
//!
//! See `docs/specs/2026-09-26-agent-kernel-design.md` for the full design.

mod approval;
mod bot;
mod model_client;
mod types;

#[cfg(test)]
mod test_support;

pub use approval::ApprovalChannel;
pub use bot::Bot;
pub use model_client::ModelClient;
pub use types::{
    ApprovalDecision, ApprovalRequest, BotError, BotOutcome, Capability, KernelError, ModelError,
    Task, TaskId, TaskOutput,
};
```

- [ ] **Step 4: Run the full test suite to verify everything still compiles and passes**

Run: `cd crates/rena-kernel && cargo test`
Expected: PASS (2 tests — the traits and test doubles compile but aren't exercised yet)

- [ ] **Step 5: Commit**

```bash
git add crates/rena-kernel/src/bot.rs crates/rena-kernel/src/model_client.rs crates/rena-kernel/src/approval.rs crates/rena-kernel/src/test_support.rs crates/rena-kernel/src/lib.rs
git commit -m "feat(kernel): add Bot/ModelClient/ApprovalChannel traits and test doubles"
```

---

### Task 3: Kernel skeleton — registration + happy-path routing

**Files:**
- Create: `crates/rena-kernel/src/kernel.rs`
- Modify: `crates/rena-kernel/src/lib.rs`

- [ ] **Step 1: Write the failing tests**

Create `crates/rena-kernel/src/kernel.rs`:

```rust
use std::collections::HashMap;
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Arc;
use std::time::Duration;

use tokio::sync::RwLock;

use crate::approval::ApprovalChannel;
use crate::model_client::ModelClient;
use crate::types::{Capability, KernelError, Task, TaskOutput};

struct CapabilityEntry {
    bots: Vec<Arc<dyn crate::bot::Bot>>,
    next_index: AtomicUsize,
}

pub struct KernelConfig {
    pub max_delegation_depth: u8,
    pub approval_timeout: Duration,
    pub max_retries: u32,
}

impl Default for KernelConfig {
    fn default() -> Self {
        KernelConfig {
            max_delegation_depth: 4,
            approval_timeout: Duration::from_secs(60 * 60 * 24),
            max_retries: 3,
        }
    }
}

pub struct Kernel {
    registry: RwLock<HashMap<Capability, CapabilityEntry>>,
    models: Arc<dyn ModelClient>,
    approvals: Arc<dyn ApprovalChannel>,
    config: KernelConfig,
}

impl Kernel {
    pub fn new(models: Arc<dyn ModelClient>, approvals: Arc<dyn ApprovalChannel>) -> Self {
        Self::with_config(models, approvals, KernelConfig::default())
    }

    pub fn with_config(
        models: Arc<dyn ModelClient>,
        approvals: Arc<dyn ApprovalChannel>,
        config: KernelConfig,
    ) -> Self {
        Kernel {
            registry: RwLock::new(HashMap::new()),
            models,
            approvals,
            config,
        }
    }

    pub async fn register_bot(&self, bot: Arc<dyn crate::bot::Bot>) {
        let mut registry = self.registry.write().await;
        for capability in bot.capabilities() {
            registry
                .entry(capability.clone())
                .or_insert_with(|| CapabilityEntry {
                    bots: Vec::new(),
                    next_index: AtomicUsize::new(0),
                })
                .bots
                .push(bot.clone());
        }
    }

    async fn pick_bot(
        &self,
        capability: &Capability,
    ) -> Result<Arc<dyn crate::bot::Bot>, KernelError> {
        let registry = self.registry.read().await;
        let entry = registry
            .get(capability)
            .filter(|e| !e.bots.is_empty())
            .ok_or_else(|| KernelError::NoCapableBot(capability.clone()))?;
        let idx = entry.next_index.fetch_add(1, Ordering::SeqCst) % entry.bots.len();
        Ok(entry.bots[idx].clone())
    }

    pub async fn submit_task(&self, task: Task) -> Result<TaskOutput, KernelError> {
        let bot = self.pick_bot(&task.capability).await?;
        let outcome = bot.handle(task, self.models.as_ref()).await;
        match outcome {
            crate::types::BotOutcome::Done(output) => Ok(output),
            crate::types::BotOutcome::Failed(err) => Err(KernelError::BotFailed(err.0)),
            other => Err(KernelError::BotFailed(format!(
                "unhandled outcome in skeleton kernel: {other:?}"
            ))),
        }
    }
}

#[cfg(test)]
mod tests {
    use std::sync::Arc;

    use super::*;
    use crate::test_support::{MockModelClient, StubApproval, StubApprovalChannel, TestBot, TestBotBehavior};

    fn kernel() -> Kernel {
        Kernel::new(
            Arc::new(MockModelClient),
            Arc::new(StubApprovalChannel {
                behavior: StubApproval::Approve,
            }),
        )
    }

    #[tokio::test]
    async fn submits_task_to_registered_bot() {
        let kernel = kernel();
        let bot = TestBot::new(
            "echo",
            "echo",
            TestBotBehavior::Succeed {
                result: serde_json::json!({"ok": true}),
                confidence: 0.9,
            },
        );
        kernel.register_bot(Arc::new(bot)).await;

        let output = kernel
            .submit_task(Task::new("echo", serde_json::json!(null)))
            .await
            .expect("task should succeed");

        assert_eq!(output.result, serde_json::json!({"ok": true}));
        assert_eq!(output.confidence, 0.9);
    }

    #[tokio::test]
    async fn unknown_capability_returns_error() {
        let kernel = kernel();
        let result = kernel
            .submit_task(Task::new("nonexistent", serde_json::json!(null)))
            .await;
        assert!(matches!(result, Err(KernelError::NoCapableBot(_))));
    }
}
```

- [ ] **Step 2: Wire `kernel` module into `lib.rs`**

Add `mod kernel;` to `crates/rena-kernel/src/lib.rs` (alongside the other `mod` lines) and add `Kernel, KernelConfig` to the `pub use types::{...}` block's neighboring re-export — add this new line right after the `mod` declarations:

```rust
pub use kernel::{Kernel, KernelConfig};
```

- [ ] **Step 3: Run the tests to verify they pass**

Run: `cd crates/rena-kernel && cargo test kernel::tests`
Expected: PASS (2 tests)

- [ ] **Step 4: Commit**

```bash
git add crates/rena-kernel/src/kernel.rs crates/rena-kernel/src/lib.rs
git commit -m "feat(kernel): add Kernel skeleton with registration and happy-path routing"
```

---

### Task 4: Round-robin routing across multiple bots

**Files:**
- Modify: `crates/rena-kernel/src/kernel.rs` (tests only — routing logic already round-robins from Task 3)

- [ ] **Step 1: Write the failing test**

Add to the `mod tests` block in `crates/rena-kernel/src/kernel.rs`:

```rust
    #[tokio::test]
    async fn round_robins_across_bots_sharing_a_capability() {
        let kernel = kernel();
        kernel
            .register_bot(Arc::new(TestBot::new(
                "bot-a",
                "greet",
                TestBotBehavior::Succeed {
                    result: serde_json::json!({"by": "a"}),
                    confidence: 1.0,
                },
            )))
            .await;
        kernel
            .register_bot(Arc::new(TestBot::new(
                "bot-b",
                "greet",
                TestBotBehavior::Succeed {
                    result: serde_json::json!({"by": "b"}),
                    confidence: 1.0,
                },
            )))
            .await;

        let first = kernel
            .submit_task(Task::new("greet", serde_json::json!(null)))
            .await
            .unwrap();
        let second = kernel
            .submit_task(Task::new("greet", serde_json::json!(null)))
            .await
            .unwrap();
        let third = kernel
            .submit_task(Task::new("greet", serde_json::json!(null)))
            .await
            .unwrap();

        assert_eq!(first.result, serde_json::json!({"by": "a"}));
        assert_eq!(second.result, serde_json::json!({"by": "b"}));
        assert_eq!(third.result, serde_json::json!({"by": "a"}));
    }
```

- [ ] **Step 2: Run the test**

Run: `cd crates/rena-kernel && cargo test round_robins_across_bots_sharing_a_capability`
Expected: PASS — `pick_bot`'s `fetch_add` + modulo already implements this from Task 3.

- [ ] **Step 3: Commit**

```bash
git add crates/rena-kernel/src/kernel.rs
git commit -m "test(kernel): verify round-robin routing across bots sharing a capability"
```

---

### Task 5: Panic supervision with bounded backoff

**Files:**
- Modify: `crates/rena-kernel/src/kernel.rs`

- [ ] **Step 1: Write the failing test**

Add to `mod tests`:

```rust
    #[tokio::test]
    async fn panicking_bot_is_retried_then_marked_degraded() {
        let kernel = Kernel::with_config(
            Arc::new(MockModelClient),
            Arc::new(StubApprovalChannel {
                behavior: StubApproval::Approve,
            }),
            KernelConfig {
                max_retries: 2,
                ..KernelConfig::default()
            },
        );
        kernel
            .register_bot(Arc::new(TestBot::new(
                "boom",
                "explode",
                TestBotBehavior::Panic,
            )))
            .await;

        let result = kernel
            .submit_task(Task::new("explode", serde_json::json!(null)))
            .await;

        assert!(matches!(result, Err(KernelError::Degraded(_))));
    }
```

Run: `cd crates/rena-kernel && cargo test panicking_bot_is_retried_then_marked_degraded`
Expected: FAIL — `submit_task` currently calls `bot.handle` directly with no panic isolation, so the whole test process aborts on panic instead of returning `Degraded`.

- [ ] **Step 2: Replace the direct `bot.handle` call with a supervised dispatch**

In `crates/rena-kernel/src/kernel.rs`, replace the `submit_task` method with:

```rust
    pub async fn submit_task(&self, task: Task) -> Result<TaskOutput, KernelError> {
        let bot = self.pick_bot(&task.capability).await?;
        let outcome = self.run_with_supervision(bot, task).await?;
        match outcome {
            crate::types::BotOutcome::Done(output) => Ok(output),
            crate::types::BotOutcome::Failed(err) => Err(KernelError::BotFailed(err.0)),
            other => Err(KernelError::BotFailed(format!(
                "unhandled outcome: {other:?}"
            ))),
        }
    }

    async fn run_with_supervision(
        &self,
        bot: Arc<dyn crate::bot::Bot>,
        task: Task,
    ) -> Result<crate::types::BotOutcome, KernelError> {
        let mut attempt: u32 = 0;
        loop {
            let bot = bot.clone();
            let task = task.clone();
            let models = self.models.clone();
            let handle = tokio::spawn(async move { bot.handle(task, models.as_ref()).await });

            match handle.await {
                Ok(outcome) => return Ok(outcome),
                Err(join_err) => {
                    attempt += 1;
                    tracing::warn!(
                        capability = %task.capability,
                        attempt,
                        error = %join_err,
                        "bot task panicked"
                    );
                    if attempt > self.config.max_retries {
                        return Err(KernelError::Degraded(task.capability.clone()));
                    }
                    let backoff = Duration::from_millis(10 * 2u64.pow(attempt));
                    tokio::time::sleep(backoff).await;
                }
            }
        }
    }
```

- [ ] **Step 3: Run the test to verify it passes**

Run: `cd crates/rena-kernel && cargo test panicking_bot_is_retried_then_marked_degraded`
Expected: PASS

- [ ] **Step 4: Run the full suite to check nothing else broke**

Run: `cd crates/rena-kernel && cargo test`
Expected: PASS (all tests so far)

- [ ] **Step 5: Commit**

```bash
git add crates/rena-kernel/src/kernel.rs
git commit -m "feat(kernel): supervise bot dispatch with panic isolation and bounded backoff"
```

---

### Task 6: Delegation with a max-depth guard

**Files:**
- Modify: `crates/rena-kernel/src/kernel.rs`

- [ ] **Step 1: Write the failing test**

Add to `mod tests`:

```rust
    #[tokio::test]
    async fn delegation_loop_hits_depth_guard() {
        let kernel = kernel();
        kernel
            .register_bot(Arc::new(TestBot::new(
                "looper",
                "loop",
                TestBotBehavior::DelegateTo("loop".to_string()),
            )))
            .await;

        let result = kernel
            .submit_task(Task::new("loop", serde_json::json!(null)))
            .await;

        assert!(matches!(
            result,
            Err(KernelError::DelegationLimitExceeded(_))
        ));
    }
```

Run: `cd crates/rena-kernel && cargo test delegation_loop_hits_depth_guard`
Expected: FAIL — `submit_task` doesn't handle `BotOutcome::Delegate` yet (falls into the catch-all `other` arm as a `BotFailed`, not `DelegationLimitExceeded`).

- [ ] **Step 2: Handle delegation in `submit_task`**

Replace `submit_task` again:

```rust
    pub async fn submit_task(&self, task: Task) -> Result<TaskOutput, KernelError> {
        if task.delegation_depth > self.config.max_delegation_depth {
            return Err(KernelError::DelegationLimitExceeded(task.capability.clone()));
        }

        let bot = self.pick_bot(&task.capability).await?;
        let outcome = self.run_with_supervision(bot, task.clone()).await?;

        match outcome {
            crate::types::BotOutcome::Done(output) => Ok(output),
            crate::types::BotOutcome::Failed(err) => Err(KernelError::BotFailed(err.0)),
            crate::types::BotOutcome::Delegate {
                to_capability,
                task: mut next_task,
            } => {
                next_task.capability = to_capability;
                next_task.delegation_depth = task.delegation_depth + 1;
                Box::pin(self.submit_task(next_task)).await
            }
            other => Err(KernelError::BotFailed(format!(
                "unhandled outcome: {other:?}"
            ))),
        }
    }
```

- [ ] **Step 3: Run the test to verify it passes**

Run: `cd crates/rena-kernel && cargo test delegation_loop_hits_depth_guard`
Expected: PASS

- [ ] **Step 4: Run the full suite**

Run: `cd crates/rena-kernel && cargo test`
Expected: PASS (all tests so far)

- [ ] **Step 5: Commit**

```bash
git add crates/rena-kernel/src/kernel.rs
git commit -m "feat(kernel): handle single-hop delegation with a max-depth guard"
```

---

### Task 7: Human-in-the-loop approval (fail-closed)

**Files:**
- Modify: `crates/rena-kernel/src/kernel.rs`

- [ ] **Step 1: Write the three failing tests**

Add to `mod tests`:

```rust
    #[tokio::test]
    async fn approved_request_resumes_the_task() {
        let kernel = Kernel::with_config(
            Arc::new(MockModelClient),
            Arc::new(StubApprovalChannel {
                behavior: StubApproval::Approve,
            }),
            KernelConfig::default(),
        );
        kernel
            .register_bot(Arc::new(TestBot::new(
                "risky",
                "risky-op",
                TestBotBehavior::NeedsApproval {
                    request: ApprovalRequest {
                        summary: "about to do a risky thing".to_string(),
                        payload: serde_json::json!(null),
                        risk_note: Some("irreversible".to_string()),
                    },
                    resume_result: serde_json::json!({"done": true}),
                },
            )))
            .await;

        let output = kernel
            .submit_task(Task::new("risky-op", serde_json::json!(null)))
            .await
            .expect("approved task should succeed");

        assert_eq!(output.result, serde_json::json!({"done": true}));
    }

    #[tokio::test]
    async fn rejected_request_surfaces_kernel_error() {
        let kernel = Kernel::with_config(
            Arc::new(MockModelClient),
            Arc::new(StubApprovalChannel {
                behavior: StubApproval::Reject(Some("too risky".to_string())),
            }),
            KernelConfig::default(),
        );
        kernel
            .register_bot(Arc::new(TestBot::new(
                "risky",
                "risky-op",
                TestBotBehavior::NeedsApproval {
                    request: ApprovalRequest {
                        summary: "about to do a risky thing".to_string(),
                        payload: serde_json::json!(null),
                        risk_note: None,
                    },
                    resume_result: serde_json::json!({"done": true}),
                },
            )))
            .await;

        let result = kernel
            .submit_task(Task::new("risky-op", serde_json::json!(null)))
            .await;

        assert!(matches!(result, Err(KernelError::ApprovalDenied(_))));
    }

    #[tokio::test]
    async fn approval_timeout_fails_closed() {
        let kernel = Kernel::with_config(
            Arc::new(MockModelClient),
            Arc::new(StubApprovalChannel {
                behavior: StubApproval::Never,
            }),
            KernelConfig {
                approval_timeout: Duration::from_millis(50),
                ..KernelConfig::default()
            },
        );
        kernel
            .register_bot(Arc::new(TestBot::new(
                "risky",
                "risky-op",
                TestBotBehavior::NeedsApproval {
                    request: ApprovalRequest {
                        summary: "about to do a risky thing".to_string(),
                        payload: serde_json::json!(null),
                        risk_note: None,
                    },
                    resume_result: serde_json::json!({"done": true}),
                },
            )))
            .await;

        let result = kernel
            .submit_task(Task::new("risky-op", serde_json::json!(null)))
            .await;

        assert!(matches!(result, Err(KernelError::ApprovalDenied(_))));
    }
```

Add `ApprovalRequest` to the `use crate::types::{...}` import at the top of the test module if not already present (it is exported from `types`, already imported via `super::*` if you re-exported it there — otherwise add `use crate::types::ApprovalRequest;` explicitly next to the other test imports).

Run: `cd crates/rena-kernel && cargo test approved_request_resumes_the_task rejected_request_surfaces_kernel_error approval_timeout_fails_closed`
Expected: FAIL — `submit_task` doesn't handle `BotOutcome::NeedsApproval` yet.

- [ ] **Step 2: Handle `NeedsApproval` in `submit_task`**

Replace `submit_task` once more:

```rust
    pub async fn submit_task(&self, task: Task) -> Result<TaskOutput, KernelError> {
        if task.delegation_depth > self.config.max_delegation_depth {
            return Err(KernelError::DelegationLimitExceeded(task.capability.clone()));
        }

        let bot = self.pick_bot(&task.capability).await?;
        let outcome = self.run_with_supervision(bot, task.clone()).await?;

        match outcome {
            crate::types::BotOutcome::Done(output) => Ok(output),
            crate::types::BotOutcome::Failed(err) => Err(KernelError::BotFailed(err.0)),
            crate::types::BotOutcome::Delegate {
                to_capability,
                task: mut next_task,
            } => {
                next_task.capability = to_capability;
                next_task.delegation_depth = task.delegation_depth + 1;
                Box::pin(self.submit_task(next_task)).await
            }
            crate::types::BotOutcome::NeedsApproval { request, resume } => {
                let decision = tokio::time::timeout(
                    self.config.approval_timeout,
                    self.approvals.request(request),
                )
                .await
                .unwrap_or(crate::types::ApprovalDecision::Rejected {
                    reason: Some("approval timed out".to_string()),
                });

                match decision {
                    crate::types::ApprovalDecision::Approved => {
                        let bot = self.pick_bot(&resume.capability).await?;
                        let outcome = self.run_with_supervision(bot, resume).await?;
                        match outcome {
                            crate::types::BotOutcome::Done(output) => Ok(output),
                            other => Err(KernelError::BotFailed(format!(
                                "unexpected outcome after approval resume: {other:?}"
                            ))),
                        }
                    }
                    crate::types::ApprovalDecision::Rejected { reason } => {
                        Err(KernelError::ApprovalDenied(reason))
                    }
                }
            }
        }
    }
```

- [ ] **Step 3: Run the three tests to verify they pass**

Run: `cd crates/rena-kernel && cargo test approved_request_resumes_the_task rejected_request_surfaces_kernel_error approval_timeout_fails_closed`
Expected: PASS (3 tests)

- [ ] **Step 4: Run the full suite**

Run: `cd crates/rena-kernel && cargo test`
Expected: PASS (all tests so far)

- [ ] **Step 5: Commit**

```bash
git add crates/rena-kernel/src/kernel.rs
git commit -m "feat(kernel): add fail-closed human-in-the-loop approval handling"
```

---

### Task 8: Concurrency test, crate docs, final verification, push

**Files:**
- Modify: `crates/rena-kernel/src/kernel.rs`
- Modify: `crates/rena-kernel/src/lib.rs`

- [ ] **Step 1: Write the failing test**

Add to `mod tests`:

```rust
    #[tokio::test]
    async fn concurrent_submissions_do_not_interfere() {
        let kernel = kernel();
        kernel
            .register_bot(Arc::new(TestBot::new(
                "alpha",
                "alpha-op",
                TestBotBehavior::Succeed {
                    result: serde_json::json!({"who": "alpha"}),
                    confidence: 1.0,
                },
            )))
            .await;
        kernel
            .register_bot(Arc::new(TestBot::new(
                "beta",
                "beta-op",
                TestBotBehavior::Succeed {
                    result: serde_json::json!({"who": "beta"}),
                    confidence: 1.0,
                },
            )))
            .await;

        let (a, b) = tokio::join!(
            kernel.submit_task(Task::new("alpha-op", serde_json::json!(null))),
            kernel.submit_task(Task::new("beta-op", serde_json::json!(null)))
        );

        assert_eq!(a.unwrap().result, serde_json::json!({"who": "alpha"}));
        assert_eq!(b.unwrap().result, serde_json::json!({"who": "beta"}));
    }
```

Run: `cd crates/rena-kernel && cargo test concurrent_submissions_do_not_interfere`
Expected: PASS (the routing/dispatch design from earlier tasks is already concurrency-safe — this test proves it, it shouldn't require new production code)

- [ ] **Step 2: Add crate-level usage docs**

Prepend to the top of `crates/rena-kernel/src/lib.rs` (above the existing `//!` line), replacing the single doc line with:

```rust
//! Agent Kernel: a generic orchestration primitive for RenaMind bots.
//!
//! Register `Bot` implementations by capability, then call
//! [`Kernel::submit_task`] with a [`Task`] naming the capability it needs.
//! The kernel routes to a registered bot, supervises it through panics with
//! bounded retry/backoff, follows single-hop delegation up to a depth
//! guard, and pauses for human approval (failing closed on timeout) when a
//! bot asks for it.
//!
//! See `docs/specs/2026-09-26-agent-kernel-design.md` for the full design
//! and explicit v1 non-goals.
```

- [ ] **Step 3: Run the full test suite one more time**

Run: `cd crates/rena-kernel && cargo test`
Expected: PASS (all tests)

- [ ] **Step 4: Run a release build to catch anything `cargo test`'s dev profile might not**

Run: `cd crates/rena-kernel && cargo build --release`
Expected: `Compiling rena-kernel...` then `Finished` with no errors

- [ ] **Step 5: Commit**

```bash
git add crates/rena-kernel/src/kernel.rs crates/rena-kernel/src/lib.rs
git commit -m "test(kernel): verify concurrent submissions don't interfere; add crate docs"
```

- [ ] **Step 6: Push everything to GitHub**

```bash
git push origin main
```

Expected: push succeeds, `origin/main` now matches local `main`.

- [ ] **Step 7: Verify the remote has everything before any local cleanup**

```bash
git status --short
git log --oneline -1
git rev-parse HEAD
git rev-parse origin/main
```

Expected: `git status --short` prints nothing (clean tree), and the two `rev-parse` commands print the same commit hash.

---

## Spec coverage check (self-review)

- Purpose/non-goals → enforced by what's built (no persistence, mock `ModelClient`/`ApprovalChannel`, no RBAC, single-hop delegation only) — Tasks 1-8.
- Concurrency architecture (A: raw Tokio, no framework) → Task 5.
- Core types → Task 1.
- Routing (capability + round-robin) → Tasks 3-4.
- Fault handling / supervision → Task 5.
- Delegation + depth guard → Task 6.
- Human-in-the-loop, fail-closed → Task 7.
- Persistence boundary (explicitly none) → no task adds it; confirmed absent by design.
- Testing plan (panic/restart, delegation guard, round-robin, concurrency, approval x3) → Tasks 4-8, one test each.
