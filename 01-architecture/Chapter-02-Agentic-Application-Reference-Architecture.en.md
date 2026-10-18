# Chapter 2 — Agentic Application Reference Architecture

> **Translated and distilled** from 第 2 章《Agentic Application 参考架构》 in `sadam-m/ai-agent-handbook`
> (forked from `aliyun/ai-agent-handbook`, Alibaba Cloud enterprise white paper).
> Distillation preserves every framework, definition, and table from the original; prose is condensed.
> Original (Chinese): [`01-architecture/第 2 章　Agentic Application 参考架构.md`](./第%202%20章　Agentic%20Application%20参考架构.md)

---

## The core argument

Chapter 1 defined the Agent as **Model + Harness**: the model is the cognitive core, but reliability can never be entrusted to the model itself. An agent needs task autonomy *and* a deterministic system constraining its capabilities, environment, and blast radius.

That reframes the architecture problem: enterprises are no longer designing *a model call*, but a complete system spanning cognition, state, execution, control, and continuous improvement.

The hard part of architecture is **drawing responsibility boundaries**, not listing components. Four boundary confusions stay hidden in demos and explode in production (longer tasks, more tools, more tenants):

1. **Model vs Harness** blurred → every model upgrade becomes an application rewrite.
2. **Orchestration logic vs Runtime** blurred → task semantics get hard-bound to infrastructure.
3. **Context vs State** blurred → incomplete conversation history gets mistaken for recoverable task facts.
4. **Sandbox, permissions, and tool descriptions** mixed together → "the model *knows about* a tool" gets mistaken for "the model *is authorized to execute* it."

This chapter gives a vendor-neutral enterprise reference architecture. It does **not** require every app to build all components — it asks you to identify business constraints first, then choose a **minimum-sufficient architecture** matched to the task's risk.

---

## 2.1 The architecture panorama

### 2.1.1 Three views of one system

Three orthogonal views describe different sides of the *same* system — they are not interchangeable architectures:

| View | Answers | Value (what failure it prevents) |
|---|---|---|
| **Component view** | What is the system made of? (Model, Harness orchestration, Context, State, Memory, Knowledge, Skill, Tool, Runtime, Sandbox, Gateway, Observability, Evaluation, Security…) | Avoids capability gaps — especially treating context management, state persistence, and result verification as "implementation details." |
| **Platform view** | Where do these components run, and who owns them? (execution plane, data & resource plane, control plane) | Avoids responsibility confusion — the same capability built by the app team vs. provided by a shared platform has very different cost, scalability, and governance. |
| **Lifecycle view** | How does the system go from architecture decision to operation, and get governed and improved while running? (Architect → Build → Run → Govern → Tune) | Avoids mistaking a one-off demo for a sustainably operated enterprise system. |

The three views cannot be derived from each other: complete components ≠ clear responsibilities ≠ controlled change. When discussing any capability, name which view you're in.

### 2.1.2 Five capability domains (component view)

1. **Business & Application layer** — users, business goals, interaction modes, task organization, acceptance criteria. Triggered by chat, API, message, timer, or business event; executed by a single agent, a long-horizon agent, multi-agent collaboration, or hybrid with workflows. This layer defines *why the system exists*, not generic agent capabilities.
2. **Agent Build & Orchestration layer** — composes Model, harness orchestration logic, context strategy, Memory, Knowledge, Skill, and Tool/Protocol/Connector definitions into a task loop that observes, judges, acts, and accepts feedback. "Build" is not one-time: the orchestration keeps working on every task run. External systems, browsers, shells, and code execution are *not* in this layer — it only describes and binds capabilities; actual execution and blast radius belong to the runtime layer.
3. **Production Runtime layer** — cross-time, cross-request task execution: scheduling, state persistence, workspaces, isolation, recovery, traffic governance, resource management.
4. **Governance & Control layer** — who may publish/run/modify agents, which resources are accessible, how versions flow, how policy is distributed, enforced, and audited. Runtime answers "can the task run stably"; governance answers "which versions and actions are *allowed* to happen."
5. **Tuning layer** — turns Traces, Metrics, Logs, costs, results, and feedback into problems located via evaluation, security analysis, and simulation — producing candidate changes *with verification evidence*. It never modifies production agents directly; changes flow back to the Build layer as new versions and pass governance gates before reaching Runtime.

**Security cuts across all five** — it is not a single component: business layer sets risk/acceptance boundaries, build layer implements constraints and verification, runtime provides isolation and execution control, governance defines identity/policy/audit, tuning validates via security evaluation and red-teaming.

### 2.1.3 Five enterprise design principles

Distilled from public architecture and engineering practice (AWS, Google, Microsoft, Anthropic, LangChain, OpenAI, H2 2025+):

1. **Model ↔ Harness decoupling.** The model is a replaceable cognitive core; the harness is the engineering shell tied to task, environment, and org requirements. A model upgrade can silently invalidate compensating prompts, tools, or loop controls — so evaluate model + harness *together*, but never bind them in implementation.
2. **Orchestration ↔ Runtime decoupling.** Orchestration defines how the agent thinks and acts; Runtime defines under what resources and failure conditions tasks keep running. Separate them so you can switch managed/self-hosted runtimes without rewriting task logic, and resume the same task when the runtime moves processes, nodes, or regions.
3. **State externalized; execution instances replaceable.** Sessions, plans, todos, tool results, workspaces, memories, and artifacts must not live only in model context or one runtime's memory. Durable events, checkpoints, and external workspaces are prerequisites for long-task recovery and distributed execution.
4. **Autonomy must be matched with permissions and verifiability.** More tools, network, data, and credentials = larger blast radius. Bound it with least privilege, short-lived credentials, egress control, sandbox isolation, result verification, and high-risk action approval.
5. **Observability, security, and evaluation are prerequisites, not patches.** Agent output is a function of model × orchestration × tools × context × budget × environment. Without recorded trajectories and environment outcomes, you can neither locate problems, reproduce evaluations, nor prove a version actually improved.

These don't demand building big — they demand that **every granted autonomy has matching boundaries, evidence, and verification**, and that anything deferred is deferred *explicitly*, with the conditions for adding it named.

---

## 2.2 Build & orchestration: Model, Harness, Context, capabilities

### 2.2.1 Model — a replaceable, routable cognitive core

The model handles understanding, reasoning, planning, structured generation, and tool-call decisions. But model selection is a *continuous* decision tied to task, harness, and runtime strategy — not a one-time vendor pick. Evaluate on five dimensions:

1. Base capability on the target domain/task type.
2. Instruction-following, information filtering, and interference resistance in long context.
3. Tool-parameter generation, tool-result comprehension, error correction, multi-step call stability.
4. Latency, price, context cost, concurrency limits, regional availability.
5. Data retention, privacy, compliance, privatization, operational boundaries.

(1–3) decide whether the agent *can* do the task; (4–5) decide whether it can run *at enterprise scale*.

Complex systems needn't use one model for everything — planning, execution, summarization, retrieval, review, and moderation have different capability/latency/cost needs. A **Model Router** can route by task type, budget, data boundary, and load. But every route and fallback changes agent behavior, so it must be **regression-tested with the full harness** — protocol compatibility is not capability equivalence. Watch for this failure mode: a fallback model returns *structurally valid* tool calls while silently dropping intermediate constraints across multi-step tasks.

### 2.2.2 Harness orchestration — organizing probabilistic judgment into controlled execution

"Broad Harness" = everything outside the model (runtime, sandbox, observability, governance — cf. LangChain's "everything but the model," Microsoft's "runtime scaffolding"). At implementation granularity that term becomes un-ownable, so the chapter narrows it:

> **The Harness orchestration layer is the code, configuration, and execution logic sitting between the model and the runtime environment — constructing context, organizing the task loop, providing capabilities, constraining behavior, and verifying intermediate results.**

It is the nerve center and the responsibility boundary: it decides **what the model sees, what it may call, when it stops, and which verifications results pass before delivery.**

A production-grade orchestration layer needs these seven capabilities (third column: the failure mode that makes each non-optional):

| Capability | Responsibility | Failure mode if missing |
|---|---|---|
| Agent loop & termination | Model calls, tool calls, result feed-back, retries, stopping, recovery, budget constraints | Tasks never converge; infinite loops / budget blowouts |
| Instruction & context engineering | Selecting/arranging system instructions, task briefs, history, tool descriptions, memory, retrieval results, environment state | Key information drowned; model drifts off-target in long context |
| Planning & delegation | Todo lists, phase goals, sub-agents, parallel execution, handoffs, result aggregation | Complex tasks lack phasing; failures can't be located or resumed |
| Tool & skill management | Capability registration, progressive disclosure, parameter validation, result truncation, large-result offloading, tool-failure classification | Tool descriptions eat the context; tool errors misread as model errors |
| Middleware & policy hooks | Permissions, approvals, audit, content filtering, custom logic around model calls, tool calls, state changes, outputs | Policy lives only in prompts — the model can bypass constraints |
| Compaction & continuation | Dialogue compression, context resets, file-based handoffs, cross-window task continuation | Long tasks die at context exhaustion, or compression corrupts task facts |
| Steering, HITL & verification | Mid-run goal correction, pre-action approval, pre-delivery tests/reviews/completeness checks | High-risk actions unsupervised; tasks "look done" but aren't trustworthy |

Two evidence points on evolution: Anthropic found that scaffolding built to compensate for *weaker* models can *constrain stronger* ones — re-evaluate and even simplify scaffolding as models improve. LangChain's practice shows that with a fixed model, trace analysis, error clustering, pre-completion verification, and loop detection can materially change task quality. **The orchestration layer is itself an evaluable object** — never design-once, never "more complex = better."

**Orchestration vs Runtime split** (decide at architecture time): orchestration owns *semantic* decisions (what next, is approval needed, how to rebuild context, is the task done); Runtime owns *physical* guarantees (which instance runs it, queueing/throttling, which checkpoint to resume after a crash, resource reclamation). Their interface should be **tasks and state**, not function-call internals — if orchestration depends on a runtime's memory structures, principle 2 is already violated.

### 2.2.3 Context, State, Memory, Knowledge, Skill — don't put everything in one context

The context window is what the model sees *in this call*; the system must manage far more. Five distinct objects:

| Object | Core question | Typical content | Lifecycle & consistency |
|---|---|---|---|
| **Context** | What should this model call see? | System prompt, task brief, history summary, tool descriptions, retrieval results, current environment state | Per-call; continuously rebuilt; **never a source of truth** |
| **State** | What has happened; where to resume? | Session, Task, Plan, Todo, Checkpoint, tool-call status | Task-scoped; needs strongly consistent, persisted facts |
| **Memory** | Which past interaction experience helps now? | User preferences, past decisions, cross-session facts, task experience | Cross-task/session; needs write policies, updates, forgetting, privacy governance |
| **Knowledge** | What external org knowledge is retrievable? | Business docs, policies, product material, RAG indexes | Synced with org content; needs permission filtering, freshness, provenance |
| **Skill** | How should this *kind* of task be done? | Instructions, scripts, templates, workflows, checklists, verifiable methods | Versioned capability assets: published, reused, rolled back |

**Memory vs Knowledge are split deliberately**: Memory is *generated by the agent at runtime* — risk is bad experience being reused forever, so it needs write thresholds and forgetting. Knowledge comes from *existing org content* — risk is permission overreach and staleness, so it needs alignment with source permission models and update cycles. One shared vector store usually loses both governances.

**Context engineering ≠ stuffing a longer window.** Select the right information at the right time: summarization, tool-result offloading, on-demand retrieval, progressive skill disclosure, sub-agent context isolation, file-based handoffs. And keep recoverable task truth *outside* model context — if compression, truncation, or model misreading can change task facts, the task loses recoverability, and long-horizon/multi-agent work becomes impossible.

### 2.2.4 Tool, Protocol, Environment — from interface calls to controlled action

- **Tool** = a capability the agent may request. **Protocol** = how capabilities are described, discovered, called, returned (Function Calling; **MCP** for tools/resources/prompts as reusable protocol capabilities; **A2A** for agent-to-agent discovery, delegation, state exchange). **Environment** = where capabilities actually take effect (Browser/Shell/Code Interpreter extend reach to systems with no API — and visibly widen blast radius).
- Their failure modes differ: tools fail on unclear descriptions/missing validation; protocols fail when *discovery* is confused with *permission*; environments fail when impact exceeds expectation.

Three inequalities to internalize:

- **Discovered ≠ authorized.** The model knowing a tool exists is not permission to run it.
- **Schema-valid parameters ≠ user-aligned intent.**
- **Sandbox success ≠ production-safe result.**

A **controlled action** therefore passes six gates, distributed across orchestration layer, Gateway, Sandbox, and control plane — if any is missing, a model's *suggestion* becomes the system's *fact*:

1. Capability selection → 2. Identity & permission check → 3. Parameter validation → 4. Isolated-environment execution → 5. Result verification → 6. State + audit record.

---

## 2.3 Production runtime & governance control

### 2.3.1 From "it runs" to "it keeps running"

A full agent loop fits in one process on a laptop. Production begins when tasks stretch to hours/days/weeks, one agent serves many users/tenants, and it holds files, network, databases, and enterprise credentials. Then the problem becomes distributed systems + security isolation + enterprise governance: intake/queue/schedule/cancel, resume after restarts and context resets, isolation of fs/commands/packages/browsers/network/secrets, per-tenant state and cost attribution, and a unified traffic entry for model/tool/agent calls. These are **architecture-phase** questions, not post-launch ops details.

### 2.3.2 Three planes

("Data plane" is renamed here to avoid confusion with business data, memory, and task state.)

| Plane | Responsibility | Manages | Key properties |
|---|---|---|---|
| **Execution plane** | Actually runs agent loops, tools, environment ops | Runtime workers, Tasks, Sessions, Queues, Sandboxes, Gateway data path | Elastic, fallible, isolated; **holds no unique ground truth** |
| **Data & resource plane** | Holds recoverable facts; provides agent-accessible capabilities | Task state, Event Log, Checkpoints, Workspaces, Artifacts, Memory, Knowledge, Models, Tools, MCP Servers | Must define consistency, ownership, retention, privacy, tenant boundaries |
| **Control plane** | Defines versions, identity, policy, deployment, quotas, improvements | Agent/Model/Tool registries, Tenants, IAM, Policy, Budgets, Deployments, Observability, Evaluation, Audit | Centrally defined, versioned, auditable, rollbackable; **decision points may be distributed** |

The control plane must **not** be a synchronous dependency of every model/tool call. The robust pattern: **policy defined centrally, enforced at distributed decision points** — orchestration middleware, Gateway, Runtime, and Sandbox judge against already-distributed policy on the request path and ship audit events back asynchronously. Cost: policy propagation delay — so high-risk policies (credential revocation, tenant bans) need forced refresh, short-lived credentials, or centralized checks as backstop.

### 2.3.3 Runtime vs Sandbox — the boundary

- **Runtime** owns the task lifecycle: turns external requests into Tasks, binds harness version + identity + tenant + budget + environment, handles queueing, concurrency, pause/cancel/timeout/retry, checkpoints, recovery. For long tasks the Runtime is a *continuous execution system*: **assume any step can be interrupted at any time, and guarantee resume from external state — never restart-from-scratch.**
- **Sandbox** bounds the blast radius: filesystem, processes, CPU/memory/storage, package installs, network, secrets injection, data exfiltration. Granularity follows risk: processes (cheap, weak) → containers (most general tasks) → microVMs/VMs/dedicated accounts (untrusted code, highly sensitive data).
- Crucially: the sandbox answers only *technical reachability*. Whether an action is **authorized** (tool permission check), **business-approved** (approval flow), and **correct** (result verification) are three separate questions owned elsewhere.

### 2.3.4 State Store, Workspace, AI Gateway

- **Don't unify the state objects.** Event Log + Checkpoint need strong consistency and ordering; Workspace needs snapshots/rollback; Artifacts need addressability/retention; Memory/Knowledge need retrievability/governance. One vector DB or one chat log forfeits most of these.
- **AI Gateway in three layers** (grouped by governance granularity — the paper's own taxonomy, not a vendor product):
  - **LLM Gateway** (per *model call*): protocol adaptation, routing, rate limiting, degradation, DR, token cost.
  - **MCP Gateway** (per *tool call*): server registry, credential hosting, tool-level ACLs, audit.
  - **Agent Gateway** (per *task*): agent routing, session continuity, tenant quotas, budgets.
  
  One gateway may implement all three — but the governed object and granularity must stay distinct, or model cost, tool permissions, and task budgets will overwrite each other in one policy blob.

### 2.3.5 Deployment topologies & selection

| Topology | Fits | Strength | Main cost |
|---|---|---|---|
| In-app SDK, direct model+tool calls | Prototypes, low risk, single app | Simple, fast, short debug chain | State, secrets, observability, policy scatter across apps |
| Managed Harness + Runtime | Fast path to production where data boundaries allow | Durable execution, isolation, scaling, observability included | Vendor lock-in, bounded extensibility, per-item compliance checks |
| Managed control plane + private execution plane | Shared management, data/tools stay in-domain | Management/execution decoupled | Cross-domain connectivity, version compat, cross-domain audit |
| Full self-hosted / hybrid Agent Platform | High compliance, large scale, deep customization | Full control of data, policy, scale, infra | Highest build + long-term ops cost; needs a platform team |

Selection rule: figure out which capabilities are your **differentiating assets** (business-specific prompts, skills, tools, eval criteria, human-collaboration flows) vs. which cross-cutting capabilities belong on a **platform** (model access, task hosting, sandbox, state, identity, observability, eval infra, security policy). Guiding principle: **the platform provides a safe, operable default path; apps keep the right to compose models, orchestration, and business capabilities.** A platform that tries to encapsulate all agent differences degenerates to lowest-common-denominator (teams route around it); a platform offering only compute solves none of the real shared problems.

---

## 2.4 Lifecycle architecture: Architect → Build → Run → Govern → Tune

### 2.4.1 Agent Release — the minimum reproducible unit

App behavior depends on far more than code: model, prompt, context, memory, knowledge, skills, tools, harness logic, budgets, and environment — any one can change the trajectory. So the releasable unit is not "the app code" but a reproducible **Agent Release** binding six element groups:

1. Model + version, parameters, inference budget, routing policy.
2. Harness orchestration code/config, loop controls, middleware, termination conditions.
3. Prompt, context policy, memory policy, skill & knowledge versions.
4. Tool / MCP / Agent capability inventory, permission policies, credential scopes.
5. Runtime, sandbox, resources, network, data-retention config.
6. Eval datasets, evaluators, baselines, admission thresholds, known risks.

Only then can you answer "which release served this production task?", reproduce a failure, and prove an improvement came from *this* change. If prompts are editable in production without versioning, every eval conclusion expires immediately.

### 2.4.2 The five-phase loop (not a waterfall)

Production facts drive new versions, with explicit paths back to architecture decisions:

| Phase | Core question | Main inputs | Main outputs | Gates / feedback |
|---|---|---|---|---|
| **Architect** | What form, how much autonomy, what boundaries for this business problem? | Goals, task structure, risk level, data sensitivity, time span, compliance | Form choice, domain split, harness/runtime boundary, permission & data boundaries, observability & eval metrics, ADRs | Architecture review, threat modeling, blast-radius assessment, minimum-sufficient judgment |
| **Build** | How to implement decisions as a reproducible Agent Release? | Architecture decisions, capability assets, models, tools, knowledge | Agent definition, orchestration, prompts, context policy, skills, tool policy, eval baselines, candidate versions | Offline eval, security checks, human review, release gates → Approved Release |
| **Run** | How to execute reliably, isolated, recoverably? | Release, tasks, identity, tenant, budget, environment | Tasks, sessions, events, checkpoints, artifacts, traces, costs, incidents | Runtime state machines, sandboxes, gateways, live policy, failure recovery |
| **Govern** | How to see, constrain, approve, attribute? | Traces, identity, tool calls, data access, costs, feedback, anomalies | Policies, approvals, audits, alerts, SLOs, risk findings, governance actions | Least privilege, online checks, behavior monitoring, human supervision, incident response |
| **Tune** | How to reproduce & confirm problems, generate improvements, ship safely? | Production traces, failures, user feedback, labels, security findings | Datasets, evaluators, experiments; prompt/context/skill/harness patches + simulation results | Regression, A/B, red-teaming, simulation, canary, approval, rollback |

Architecture is phase one *deliberately*: form choice, autonomy grants, domain splits, and permission boundaries lock in the cost range of everything after. Leaving them implicit in Build means implementation details define architecture constraints backwards — fixable only by rewrite, not patch. The dashed arrows from Govern/Tune back to Architect exist for exactly this: when the risk is in the *form or permission model itself*, redesign the architecture — don't stack more patches.

### 2.4.3 Governance is a phase *and* a cross-cutting constraint

It gets its own section because post-production observability, multi-agent identity, tool permissions, data boundaries, cost, approvals, and security ownership need unified methods and owners. But governance also takes different shapes per phase: permission models and data boundaries are set in Architect; tool policies, approval points, and audit fields are implemented in Build; sandboxes/gateways/runtimes enforce in Run; observability/security analysis detect in Govern; evaluation/simulation verify in Tune. **Self-test: if a governance requirement can only be met by post-launch process and people — never landing in architecture, interfaces, or data structures — it will likely fail at scale.**

### 2.4.4 From production evidence to controlled change

Tuning ≠ letting the agent rewrite itself in production. Any model/feedback/trace-generated patch to prompts, memory, skills, or harness is a **candidate change**: reproducible eval → security check → versioning → canary → rollback-safe release. That's why Tune flows back into Build and through gates — never straight into Run. (The Tune→Run dashed arrow is only an *observation* channel for online eval/A-B on live traffic, not a change path.)

This is also why Run must emit structured traces, state, and cost facts: no reproducible evidence → tuning runs on vibes; no versioned releases → improvements can't be attributed to changes. **Observability → evaluation → release is one chain; break any link and the loop degrades to manual knob-twiddling.**

---

## 2.5 Chapter summary (author's)

The core of an Agentic Application is not a framework, model, or managed service — it is a set of clearly-owned system capabilities that evolve independently yet collaborate through the lifecycle loop. Model = cognitive core; orchestration layer organizes and constrains behavior; Context/State/Memory/Knowledge/Skill manage information and capabilities; Tool/Protocol/Environment connect to the real world; Runtime/Sandbox/State Store/Gateway carry production execution; control plane + observability + security + evaluation close the responsibility and improvement loop.

The three views mitigate three risks: component view → capability gaps; platform view → responsibility confusion; lifecycle view → mistaking a demo for an operable system. **Architecture's goal is not bigness — it is matching every granted autonomy with commensurate execution, observability, security, and evaluation capability**, choosing the minimum-sufficient architecture for the task's predictability, blast radius, data sensitivity, time span, and business risk — then evolving on production evidence.

---

## Distiller's notes — how this chapter maps to your work

*(Ziggy's mapping, not in the original.)*

- **Your Week 1 build** is §2.2.3 in miniature: the chapter's five-way split (Context/State/Memory/Knowledge/Skill) is exactly why your assembler must never treat "everything in the window" as one blob — and why recoverable task truth lives *outside* model context.
- **The six gates of a controlled action** (§2.2.4) are the production version of the guide's tool-calling chapter — compare with Artex's intercept/guard layers you already read.
- **The three gateway layers** (§2.3.4) are a reference design for your **LLM gateway portfolio project**: the paper explicitly separates the LLM Gateway (per model call: routing, rate limits, token cost) from the MCP Gateway (per tool call) and the Agent Gateway (per task). Your MVP is the first of the three.
- **Agent Release** (§2.4.1) is the answer to "what does production-grade actually mean": version *everything* that can change behavior, or your evals prove nothing.
- **The orchestration↔runtime split** (§2.2.2, principle 2) is worth re-reading before your Week 5 durable-orchestration build.
