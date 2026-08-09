# CCAR-F Study Progress

Status legend: [ ] not started · [~] in progress · [x] complete

## Domain 1 – Agentic Architecture & Orchestration (27%)
- [x] 1.1 Agentic loops & stop_reason — API is stateless; the harness loops/executes tools/holds state, Claude only decides. Branch on `stop_reason` (`end_turn`=exit, `tool_use`=run tool & continue, `max_tokens`=truncated, `pause_turn`=resume); reply with `tool_result` matched by id, `is_error` for failures; bound the loop.
- [x] 1.2 Coordinator–subagent orchestration — Coordinator decomposes→delegates→collects→synthesizes; a subagent is an opaque tool call with isolated context that returns only a string. Context flows explicitly (down via brief, up via string); use a shared output contract for clean synthesis. Go multi-agent only for parallel/independent work or context overflow (~15× cost).
- [x] 1.3 Subagent invocation, context passing, spawning — A subagent inherits only its brief, returns only a string; spawn time is the only write window. Engineer the brief as minimal-sufficient context + a shared output contract. Parallel = multiple `tool_use` in one turn; sequential = thread A's result into B's brief. Scope tools by omitting them (omit > forbid in prose); prefer explicit spawning over auto-routing.
- [x] 1.4 Multi-step workflows: enforcement & handoff — Match enforcement to stakes: high-stakes/irreversible ordering needs structural gating (which tools exist per harness state), low-stakes can use prompt guidance (lowers but never removes error). Gating ≠ `tool_choice` (selection among existing tools) — orthogonal. Hand off step state via harness + structured output, not the transcript; on failure surface `is_error`, then retry/escalate/abort and halt dependents.
- [x] 1.5 Agent SDK hooks — Deterministic harness-side callbacks at fixed lifecycle events; run every time regardless of the model's choices. PreToolUse prevents (`permissionDecision`: allow/deny/ask, argument-aware); PostToolUse reacts (format/lint, `decision:block` feeds errors back → self-correct loop); SubagentStop enforces a subagent's output contract before it reaches the coordinator; `continue:false` is the global kill switch. Choose hooks over prompts for guarantees, over tool-omission for input-aware policy; orthogonal to `tool_choice`. Shell hook (settings.json, stateless/shareable) vs SDK callback (in-process, needs app state).
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
