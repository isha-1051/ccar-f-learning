# CCAR-F Study Progress

Status legend: [ ] not started · [~] in progress · [x] complete

## Domain 1 – Agentic Architecture & Orchestration (27%)
- [x] 1.1 Agentic loops & stop_reason — Stateless/single-turn API; the *harness* loops, executes tools & carries state, Claude only decides; branch on `stop_reason` (exit on `end_turn`, `max_tokens`=truncated, `pause_turn`=resume server-side), append assistant turn + user `tool_result` (match by id), report failures via `is_error`, bound the loop.
- [x] 1.2 Coordinator–subagent orchestration — Coordinator decomposes→delegates→collects→synthesizes; a subagent is an **opaque, expensive tool call that returns a string**, each with its own isolated harness-managed context (state lives in the harness → total isolation). All coordination is explicit: seed context *down* into the brief, distilled results *up*; siblings can't see each other, coordinator is sole router. Push a **shared output contract** into every brief (prevents fragile synthesis parsing). Escalate to multi-agent ONLY for genuinely parallel/independent work or context overflow (it costs ~15× tokens) — sequential+dependent+fits-one-context → single agent (over-orchestration trap).
- [ ] 1.3 Subagent invocation, context passing, spawning
- [ ] 1.4 Multi-step workflows: enforcement & handoff
- [ ] 1.5 Agent SDK hooks
- [ ] 1.6 Task decomposition strategies
- [ ] 1.7 Session state, resumption, forking

## Domain 2 – Tool Design & MCP Integration (18%)
- [ ] 2.1 Tool interface design
- [ ] 2.2 Structured MCP error responses
- [ ] 2.3 Tool distribution & tool_choice
- [ ] 2.4 MCP server integration
- [ ] 2.5 Built-in tools (Read/Write/Edit/Bash/Grep/Glob)

## Domain 3 – Claude Code Configuration & Workflows (20%)
- [ ] 3.1 CLAUDE.md hierarchy & @import
- [ ] 3.2 Custom slash commands & skills
- [ ] 3.3 Path-specific rules
- [ ] 3.4 Plan mode vs direct execution
- [ ] 3.5 Iterative refinement
- [ ] 3.6 CI/CD integration

## Domain 4 – Prompt Engineering & Structured Output (20%)
- [ ] 4.1 Explicit criteria / false positives
- [ ] 4.2 Few-shot prompting
- [ ] 4.3 Structured output via tool_use & JSON schemas
- [ ] 4.4 Validation, retry, feedback loops
- [ ] 4.5 Batch processing
- [ ] 4.6 Multi-instance / multi-pass review

## Domain 5 – Context Management & Reliability (15%)
- [ ] 5.1 Preserving info across long context
- [ ] 5.2 Escalation & ambiguity resolution
- [ ] 5.3 Error propagation across multi-agent systems
- [ ] 5.4 Context management in large codebases
- [ ] 5.5 Human review workflows & confidence calibration
- [ ] 5.6 Information provenance & multi-source synthesis
