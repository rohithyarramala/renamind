# RenaMind Roadmap

RenaMind is an AI architecture built to close a specific gap: there is no
reliable, accountable **central intelligence** layer that coordinates
specialized agents, secures their access to tools and data, remembers what
matters across their lifetime, and stays observable and compliant while
doing it. This document lays out the vision, the design principles, the
subsystems, and the order we intend to build them in.

This is a living document. It will be revised as each subsystem gets its
own detailed design spec.

## Core focus

**Central intelligence.** One coordinating system that routes work to
specialized internal bots, not a pile of disconnected single-purpose agents
with no shared context.

**Reliability.** Async-first execution, real observability (system and
agent metrics), and a runtime built in a language that fails at compile
time instead of in production.

**Accountability.** Every output carries a confidence score. Every action
is bound to a role. Every credential lives in a vault, not in a prompt.
Nothing gets a free pass because "the AI did it."

## Design principles

- **Rust-first.** Not Python. The two hardest subsystems here — the
  secrets vault and the guardrails layer — are exactly where
  memory-unsafety becomes a security hole. Rust removes whole classes of
  that risk at compile time, and it's light enough on server resources to
  run cheaply at scale. Where a specific piece is better served by another
  fast, server-friendly language, that's a deliberate per-subsystem
  exception, not the default.
- **Async-first.** Every subsystem assumes concurrent, non-blocking
  execution as the default, not an afterthought bolted on later.
- **Provider-agnostic LLM access.** Any LLM provider is supported through
  a gateway; OpenRouter is the default/primary route so no single model
  vendor is a hard dependency.
- **Compliance by design.** Data residency, deletion, and audit-trail
  requirements (GDPR and equivalent regimes elsewhere) are a property of
  the Memory Store and Vault from the start, not a retrofit.
- **Confidence, not blind trust.** Agent and model outputs are scored, not
  taken at face value — surfaced consistently across the Agent Kernel and
  LLM Gateway.

## Subsystems

The original feature list maps onto nine concrete subsystems. Each will
get its own full design spec (brainstormed and reviewed the same way this
roadmap was) before it's built.

1. **Agent Kernel** — orchestrates multiple internal bots, each scoped to
   a specific task; owns bot lifecycle, task routing, and inter-bot
   handoff.
2. **Secrets Vault** — stores credentials and secrets with strict,
   auditable access control; designed to fail closed under compromise or
   jailbreak attempts, not just under normal use.
3. **Memory Store** — durable, backup/recoverable data store so
   intelligence survives an agent's death; supports connecting facts
   across time for future reasoning, not just raw LLM context — memory
   architecture isn't assumed to be "whatever the LLM's context window
   holds."
4. **Identity & Access (RBAC)** — solo mode (single admin) and enterprise
   mode (role-based employee access); permission boundaries that hold even
   under adversarial prompts — a role can't talk its way into a higher
   one.
5. **Tool & MCP Hub** — a pluggable way to connect new tools and MCP
   servers without touching the core.
6. **Automation Engine** — n8n-style workflow automation, both visual and
   code views, built for the reliability n8n itself is often criticized
   for lacking.
7. **Observability** — system metrics (RAM, storage, utilization) plus
   agent-level operational metrics.
8. **Guardrails** — defends the LLM I/O boundary itself against jailbreaks
   and prompt injection, distinct from Vault (secret storage) and RBAC
   (permission model).
9. **LLM Gateway** — routes to any LLM provider, OpenRouter as the
   default/primary.

**Cross-cutting, not standalone subsystems:** confidence scoring (spans
Agent Kernel + LLM Gateway) and compliance/GDPR (spans Memory Store +
Vault + Observability's audit logging).

## Build order

1. **Phase 1 — Foundation:** Agent Kernel + LLM Gateway + the async
   runtime skeleton. Nothing else works without a way to run bots and talk
   to a model.
2. **Phase 2 — Trust:** Secrets Vault + Identity & Access + Guardrails.
   Lock down who/what can do what *before* real tools or real data touch
   the system.
3. **Phase 3 — Capability:** Memory Store + Tool & MCP Hub. Once execution
   is trustworthy, give agents persistent memory and real-world reach.
4. **Phase 4 — Surface:** Automation Engine + Observability. Build the
   user-facing workflow layer and operational visibility on a core that's
   already secure and capable.

Rationale for this order: security-critical infrastructure (vault, RBAC,
guardrails) comes before any subsystem that lets agents touch real tools
or real data — never the reverse.

## Open questions

These are intentionally unresolved here — each gets its own
brainstorming/spec cycle before work starts:

- Which subsystem to spec first (Phase 1 candidates: Agent Kernel vs. LLM
  Gateway as the very first piece)
- Concrete memory backend (graph DB? vector store? hybrid?) for the Memory
  Store
- Concrete RBAC model (RBAC vs. ABAC vs. hybrid) for Identity & Access
- Rust web/async framework choices (axum/tokio, or otherwise) per
  subsystem
- Specific GDPR/compliance mechanics (data residency options, deletion
  SLAs, audit log retention)

## License

RenaMind is developed by **Renitiate Technologies** and released under
the [PolyForm Noncommercial License 1.0.0](LICENSE) plus a required
attribution term — free for noncommercial use, commercial use requires a
separate license from Renitiate Technologies.
