---
name: "harness-engineering"
description: "Apply Harness Engineering principles to build stable, production-ready AI agents. Invoke when starting a new Agent project or implementing new features that require reliable tool use, context management, feedback loops, and error handling."
---

# Harness Engineering

Harness Engineering is the discipline of building the **environmental system** that allows AI agents to be reliably controlled and stably operated in production.

**Core Formula**: `Agent = Loop(Model + Harness)`

## The Eight Elements

Every Harness system must address these eight concerns:

| # | Element | Key Question |
|---|---------|--------------|
| 1 | **Task Expression** | How is the task declared? Input/output contracts must be unambiguous. |
| 2 | **Context Organization** | What context does the agent receive? Only feed task-relevant information. |
| 3 | **Tool Governance** | How are tools discovered, validated, and intercepted? |
| 4 | **State Management** | How is state saved, restored, and pruned? |
| 5 | **Feedback Loops** | How does execution feedback flow back to the model? |
| 6 | **Error Classification** | How are errors categorized, retried, or escalated? |
| 7 | **Security Boundaries** | How are dangerous operations prevented at runtime? |
| 8 | **Verification** | How does the system verify the task was actually completed? |

## Five-Layer Architecture

### Layer 1: Context Assembly System

Prompt is not a static document — it is a **dynamic assembly system**.

Structure context by priority and freshness:

1. Base template
2. User preferences (SOUL.md)
3. Project context (AGENTS.md)
4. Skills
5. Tool descriptions
6. Tool definitions JSON

**Key principle**: Different information sources have different priorities, freshness, and responsibilities. Do not dump everything into one big prompt.

### Layer 2: Tool Governance System

Tool complexity escalates immediately. Behind every tool system are four concerns:

1. **Discovery** — How does the model find tools?
2. **Validation** — How are parameters checked?
3. **Risk Classification** — How are tools risk-leveled (read/write/execute/critical)?
4. **Interception** — How are calls uniformly intercepted and audited?

Build a tool system, not a tool list. A real tool system must be controllable, auditable, and evolvable.

### Layer 3: Security & Approval System

Once an agent can write files, execute shell commands, or modify config, it is no longer a "chat model" — it is an **operating system participant**.

Design three defensive layers:

1. **Subprocess management**: Session pools, quantity limits, output truncation, timeout termination, auto-cleanup
2. **Command guarding**: Command parsing, dangerous command blacklist, system hint relay
3. **Approval system**: Risk classification, approval modes, fingerprint caching, dangerous mode fallback

**Key principle**: Security cannot rely on "the model will behave." Real security boundaries must exist at runtime.

### Layer 4: Feedback & State System

Agents sustain continuous work not by "thinking" but by **receiving feedback**:

- Tool execution results
- Approval status
- Output truncation events
- Search result volumes
- Command rejection reasons
- Task plan updates
- Session recoverability

Translate system internals into **feedback the model can consume**. Without this translation layer, the model only sees "failed." With it, the model understands the cause and can take corrective action.

### Layer 5: Entropy Management System

Once an agent runs continuously, the system inevitably accumulates "dirt":

- Prompts grow longer
- Rules become outdated
- Context becomes bloated
- Tool definitions multiply without maintenance
- Session history becomes heavy
- Memory fills with low-value information

Long-term Harness requires ongoing **entropy management** — expire stale rules, compress long contexts, evict low-value history, maintain project knowledge in sustainable ways.

## Execution Checklist

When applying Harness Engineering to a new project or feature, verify:

### Task Definition
- [ ] Input contract is explicit (what does the agent receive?)
- [ ] Output contract is verifiable (how do we know it succeeded?)

### Context
- [ ] Context is assembled by priority, not dumped
- [ ] Only task-relevant context is fed downstream
- [ ] Noise (nav, ads, scripts) is excluded

### Tools
- [ ] Tools are declared with DSL-style definitions
- [ ] Parameters are validated before execution
- [ ] Risk levels are classified and enforced
- [ ] Calls can be intercepted and audited

### State
- [ ] Intermediate artifacts are persisted (not just final output)
- [ ] State can be recovered from checkpoints
- [ ] State is pruned when it grows stale

### Feedback
- [ ] System errors are translated into model-readable feedback
- [ ] Model understands why failures occurred, not just that they occurred

### Error Handling
- [ ] Errors are classified (network, dependency, permission, etc.)
- [ ] Retry logic exists for transient failures
- [ ] Failures escalate to the appropriate environment/capability level

### Security
- [ ] Dangerous operations are blocked at runtime, not just warned in prompts
- [ ] Subprocess execution has limits (quantity, timeout, cleanup)
- [ ] Write operations are scoped to workspace

### Verification
- [ ] Task completion is verifiable (file exists, structure matches, content is present)
- [ ] Evidence is persisted (logs, artifacts, snapshots)

## Core Philosophy

> **Harness = The environment system that makes Agent controllable and stably operable**

The most important words are not "Agent" or "Model." They are:

- **Constraint** — Agents must operate within boundaries
- **Feedback** — Agents must understand results, not just outputs
- **Stability** — Systems must be maintainable and recoverable over time

An agent's capability is not just model capability — it is `Model × Environment`.

The same model, in different systems, can produce wildly different results.

**Environment is the reusable asset. Models can be swapped; Harness cannot be skipped.**

## When to Invoke

Apply this skill when:

1. Starting a new Agent project from scratch
2. Implementing new capabilities that require tool use, context management, or state persistence
3. Designing feedback loops and error handling for agent operations
4. Establishing security boundaries for an agent that can modify files or execute commands
5. Building multi-step pipelines where intermediate state must be managed

This skill should be invoked **at project initialization**, not retrofitted later.
