# AI Engineer Curriculum — Full Stack + Agentic Engineering

**Starting point:** team is already comfortable with LLM APIs and prompting. This skips that layer and starts from RAG.
**Structure:** 12 modules, roughly in dependency order, split into three tracks that converge at the end.

---

## Track A — Making models useful with your data

### Module 1: Retrieval-Augmented Generation (RAG)
- Core architecture: retriever (vector store / BM25 / graph) + generator, and why it beats stuffing everything into a prompt or into weights
- Embeddings and vector databases — what they're good at, what they're not
- Chunking strategy: semantic chunking (split on headings/sections, not char count), parent-child retrieval (embed small chunks, return larger context)
- Hybrid retrieval (dense + BM25) + reranker as the default starting point — resist the urge to over-engineer before measuring
- Advanced patterns and when each earns its cost: Self-RAG (model critiques its own retrieval), Corrective RAG, GraphRAG (relationship-heavy queries — compliance, research synthesis), Adaptive RAG (a complexity classifier routes simple queries to cheap retrieval and complex/relational ones to agentic or graph pipelines)
- Contextual compression: summarize/filter retrieved content before it hits the prompt, not just retrieve-and-dump
- Evaluating RAG: RAGAS, context precision/recall, faithfulness — build the eval set before scaling the pipeline, not after

### Module 2: Fine-Tuning
- The decision sequence that avoids wasted effort: **Prompt → RAG → Fine-tune → Distill**. Stop at the first one that solves the problem.
- The real dividing line: RAG changes what the model *knows* (facts that change); fine-tuning changes how it *behaves* (tone, schema, refusal patterns, fixed style) — they're often complementary, not competing
- Full fine-tuning vs LoRA vs QLoRA — LoRA/QLoRA are the default for almost every team in 2026
- Technique by data type: SFT (labeled input/output pairs), DPO/ORPO/KTO (preference data), RFT (verifiable-reward tasks)
- The strongest 2026 case for fine-tuning most teams overlook: **distillation** — use a frontier model to generate high-quality outputs, fine-tune a small open model to mimic them, cut cost and latency
- Non-negotiable: the eval harness must exist *before* training, or you can't tell if a checkpoint is actually better

---

## Track B — Agentic engineering

### Module 3: Context Engineering
- Definition: designing what the model sees on every call — not just the prompt, but memory, tools, retrieval, and state
- LangChain's four operations: **Write** (scratchpads/memory files outside the active window), **Select** (retrieve only what's relevant), **Compress** (summarize before context rot sets in), **Isolate** (split work across separate context windows/sub-agents)
- **Context rot** — degradation as the window fills, now serious enough that monitoring vendors build dedicated products for it
- The **"dumb zone"** (Dex Horthy) — the middle 40–60% of a large context window is where recall degrades most; treat the window as an attention budget, not free storage

### Module 4: Loop Engineering (Agent Loops)
- The runtime formula: Agent = LLM + Memory + Planning + Tool Use
- **ReAct** — Thought → Action → Observation, interleaved reasoning and acting; good for adaptive, short-horizon tasks; weak on multi-step dependencies
- **Plan-and-Execute** — plan holistically upfront, then execute; more rigid, more predictable, needs a replanning mechanism when the initial plan is wrong
- **Reflexion / self-reflection loops** — the agent critiques its own output and revises
- **Ralph loop** — refine-until-done on a fixed task, rooted in "bash loop" thinking: keep feeding the task back until the completion criteria are met
- **Agent-Computer Interface (ACI)** — for coding agents specifically: a well-tuned interface (linter that blocks invalid edits, windowed file viewer instead of raw `cat`, concise search results) measurably beats a better prompt on the same model

### Module 5: Harness Engineering
- The equation: **Agent = Model + Harness**. If you're not the model, you're the harness.
- A harness is every piece of code, config, and execution logic around the model — tools, sandboxes, feedback loops, permission tiers, memory policies
- Distinguish: **agent harness** (one model + tools toward one task — Claude Code, Cursor, Codex are all harnesses) vs **system harness** (outer orchestration dispatching to multiple agent harnesses) vs **coding harness** (task-specific harness for software work)
- **Agentic Harness Engineering (AHE)**: treating the harness itself as measurable and revisable — tool schemas, planning artifacts, retrieval strategy, sandbox config, verification sensors, routing rules
- Most agent failures trace to missing repo context, brittle tool interfaces, or weak validators — not bad model generation. This is the highest-leverage place to spend engineering time.

### Module 6: Model Context Protocol (MCP)
- What it solves: the N×M integration problem (N tools × M clients) — MCP reduces it to N+M by giving every tool/client a single protocol to implement
- Architecture: **host, client, server**; three primitives — **tools, resources, prompts** (what the agent can do, what it can read, how it's instructed)
- Code execution with MCP: have agents write code to call tools instead of calling them directly — cuts context overhead up to ~98.7% when many servers are connected
- Governance: MCP is now vendor-neutral, donated to the Agentic AI Foundation (Linux Foundation, backed by Anthropic/Block/OpenAI/Google/Microsoft/AWS)

### Module 7: Skills
- The mental model: **MCP is the kitchen (tools/access); Skills are the recipes (how to do the task well)**
- Mechanism: **progressive disclosure** — only name+description load at session start (~100 tokens/skill); full SKILL.md instructions load only when triggered (<5k tokens)
- Design principles: progressive disclosure, composability (work alongside other skills), portability (same skill runs on Claude.ai, Claude Code, API)
- The single highest-leverage authoring skill: the description is a **routing rule**, not documentation — write it in the language a real request would use ("use when the user asks for X"), not a generic capability description
- Decision point: Skills for portable, reusable procedural knowledge; subagents for self-contained tasks needing independent workflows and restricted tool access; they compose (subagents can use skills)

### Module 8: Multi-Agent Orchestration
- Reference architecture: **orchestrator-worker** — a lead agent decomposes the task and spawns subagents with independent context windows
- The design decision that matters most: the **isolation boundary** — each subagent gets a self-contained task, no visibility into other subagents, no mid-task coordination. That's what enables true parallelism.
- Contrast: **swarm/peer-to-peer** — agents share state via a message bus; flexible but much harder to reason about and debug
- The honest cost/benefit: Anthropic's research system beat single-agent by 90.2% on complex research tasks — at ~15x token cost. Multi-agent wins on breadth-first, parallelizable problems; it's *worse* than single-agent for tightly interdependent tasks like coding.
- Caution before adopting: teams have spent months building elaborate multi-agent systems only to find better single-agent prompting matched the result. Default to single agent; earn your way into multi-agent with evidence.

### Module 9: 12-Factor Agents (Production Discipline)
- Core insight: most agents that succeed in production aren't autonomous marvels — they're well-engineered deterministic software with LLM calls placed at specific decision points
- Highest-leverage factors: own your prompts, own your context window, own your control flow, contact humans via explicit tool calls, keep the agent reducible to inspectable state transitions
- The anti-pattern to avoid: frameworks/abstractions that hide the prompts, control flow, state, or human-approval paths you need to debug

---

## Track C — Shipping and operating it

### Module 10: Evals
- Separate two things people conflate: the **agent harness** (memory, tools, everything non-model) vs the **eval harness** (the validation layer for that agent — arguably part of the harness, not separate)
- The practical ladder: deterministic/unit checks → LLM-as-judge on traced executions → continuous production monitoring
- Calibrating an LLM judge: grade ~50 examples yourself, run the judge, measure agreement — below 80%, the rubric needs work, not the model
- For anything with delegation (subagents, multi-step pipelines): eval the full delegation tree, not just the final output — a bad intermediate decision can still produce a lucky-looking final answer

### Module 11: LLMOps & Observability
- LLMOps ≠ MLOps: token cost, prompt management, and output-quality evaluation replace the classical concerns of model drift and retraining pipelines
- Tracing standard: OpenTelemetry GenAI semantic conventions; tools — Langfuse, LangSmith, Arize Phoenix, Braintrust, Helicone
- Structured logging: model version, prompt hash, token usage, latency, trace ID — not just "request completed"
- Cost control: model routing (cheap model for easy queries, frontier model for hard ones), shadow-mode/canary rollouts before full deploy
- Incident runbooks as standard practice: one per failure mode (prompt injection → analyze trace, harden guardrails; latency spike → chunked prefill, rate limits; API outage → failover)

### Module 12: Security & Guardrails
- The core vulnerability: **prompt injection** — direct (user attacks their own session) and indirect (malicious instructions embedded in retrieved/fetched content the agent trusts). OWASP Top 10 for LLM Applications is the reference taxonomy.
- Defense-in-depth, not a silver bullet: instruction hierarchy (privileged vs untrusted input), context isolation (structurally separating instructions from data), output validation, sandboxed execution, least-privilege tool access
- Assume injection eventually succeeds — design for containment (blast radius) rather than prevention alone
- For agentic systems specifically: the more tools and autonomy you grant, the bigger the blast radius of a single successful injection — sandboxing and permission tiers matter more than for a plain chatbot

---

## Suggested sequencing

Run Track A (RAG, fine-tuning) and Track B (context/loop/harness engineering) in parallel — they don't depend on each other. Converge on MCP → Skills → Multi-agent → 12-Factor once both tracks land. Close with Track C (evals, LLMOps, security) as the "what it takes to actually ship this" capstone — teams that skip straight to building agents without this track are the ones whose demos don't survive production.
