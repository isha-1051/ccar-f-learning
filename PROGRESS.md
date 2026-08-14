# CCAR-F Study Progress

Status legend: [ ] not started · [~] in progress · [x] complete

## Domain 1 – Agentic Architecture & Orchestration (27%)
- [x] 1.1 Agentic loops & stop_reason — API is stateless, so the harness loops and holds state while Claude only picks the next action via `stop_reason`.
- [x] 1.2 Coordinator–subagent orchestration — a coordinator decomposes, delegates to isolated subagents returning only strings, then synthesizes — reserved for parallel or independent work.
- [x] 1.3 Subagent invocation, context passing, spawning — a subagent gets only its brief and returns only a string, so engineer the brief and output contract when spawning.
- [x] 1.4 Multi-step workflows: enforcement & handoff — match enforcement to stakes (structural gating for irreversible ordering, prompts otherwise) and pass state via structured output, not transcript.
- [x] 1.5 Agent SDK hooks — deterministic harness callbacks at lifecycle events that always run (PreToolUse prevents, PostToolUse reacts, SubagentStop enforces contracts), chosen for guarantees.
- [x] 1.6 Task decomposition strategies — decompose along task structure (stage/data/role) only when a real trigger fires, at coarsest independent-verifiable granularity, for quality/latency never cost.
- [x] 1.7 Session state, resumption, forking — a session is the persisted `messages` array; resume continues one timeline (re-observe, world drifted), fork copies into a parallel branch.

## Domain 2 – Tool Design & MCP Integration (18%)
- [x] 2.1 Tool interface design — a tool definition is prompt engineering for a model reader: design at task altitude, enforce via schema, shape/scope results.
- [x] 2.2 Structured MCP error responses — errors are prompts: return recoverable task failures in-band via `isError` with corrective guidance.
- [x] 2.3 Tool distribution & tool_choice — match the fix to its layer: distribution scopes, descriptions guide, tool_choice forces, hooks block.
- [x] 2.4 MCP server integration — scope encodes intent, `${VARS}` keep secrets out, buy standard integrations, expose catalogs as resources.
- [x] 2.5 Built-in tools — retrieve the minimum that answers the question, at the narrowest granularity the task allows.

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
