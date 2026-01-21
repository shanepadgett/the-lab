# RLM Agent Proof-of-Loop Plan

## Intent

Build a minimal, real agent loop that demonstrates recursive context navigation (RLM-style), basic file reading and editing, and a traceable end-to-end execution inside a VS Code extension. The goal is not scale. The goal is correctness, observability, and confidence.

## Core Principles

- Context is an environment, not a static prompt. All access is explicit.
- The agent is a loop with a trace; every action produces an event.
- Tools are capability-guarded and return structured results.
- Recursion is bounded and budgeted.

## Architecture (Minimal, Real)

### Components

1) Extension Host (TypeScript)
   - Owns UI integration, tool execution, session lifecycle.
   - Spawns the RLM runtime subprocess and bridges protocol events.
   - Enforces tool policy and handles file system operations.

2) RLM Runtime (Python)
   - Implements recursive reasoning and context navigation.
   - Issues explicit context access and tool requests.
   - Produces a recursive trace of subcalls and evidence.

3) Model Adapter (Single Provider)
   - A single streaming or non-streaming model wrapper.
   - Exposes a single method for prompt + tool schemas.

### Runtime Flow (Happy Path)

1) User submits task in VS Code UI.
2) Extension Host sends new_task event to RLM runtime.
3) RLM runtime requests context via context.read/search.
4) RLM runtime performs subcall(s) on small slices.
5) RLM runtime emits tool.request for apply_patch.
6) Extension Host executes tool, emits tool.result.
7) RLM runtime validates via read_file and completes.
8) Extension Host renders completion and trace.

## Protocol (JSON Lines)

Each line is a single JSON object with a required type field.

### Required Event Types

- new_task: { id, prompt, workspace }
- context.read: { id, path, startLine?, endLine? }
- context.search: { id, query, path? }
- subcall.start: { id, parentId, intent, scope }
- subcall.end: { id, parentId, summary, citations[] }
- tool.request: { id, name, params }
- tool.result: { id, ok, output, error? }
- assistant.message: { id, content, partial? }
- completion: { id, status, summary, citations[] }
- error: { id, error, recoverable }

### Protocol Rules

- Every tool.request MUST yield exactly one tool.result.
- Every context access MUST be recorded as an event.
- subcall events MUST form a tree via parentId.
- completion MUST include citations for any final claims.

## Tools (Initial Set)

1) list_files
   - Input: { root }
   - Output: { files[] }

2) read_file
   - Input: { path, startLine?, endLine? }
   - Output: { content, startLine, endLine }

3) apply_patch
   - Input: { patch }
   - Output: { applied, summary, diff? }

Optional (later): execute_command

- Input: { command, cwd? }
- Output: { exitCode, stdout, stderr }

## RLM Runtime Design

### Context Access API

- read(path, startLine, endLine)
- search(query, path?)
- summarize(text)

### Recursive Algorithm (Minimal)

1) Analyze task, define sub-questions.
2) For each sub-question:
   - Read minimal context chunk.
   - Subcall the model on that chunk.
   - Extract claims + citations.
3) Synthesize final action with citations.
4) If editing required, emit apply_patch.

### Guardrails

- maxDepth: 2
- maxSubcalls: 6
- maxTokensPerSubcall: 1000
- maxTotalTokens: 6000
- timeoutSeconds: 60

## Minimal Demo Workspace

Create a tiny repo or folder:

- sample.ts: small utility function
- sample.test.ts: tiny test

## Demo Task (Proof)

"Add a function that normalizes whitespace in sample.ts and update its test."

Required trace:

- read_file for sample.ts
- subcall with change plan
- tool.request apply_patch
- read_file verify changes
- completion with summary + citations

## Verification Criteria

- Evidence trace shows explicit context navigation.
- One tool result per tool request.
- Patch applies cleanly.
- Final file state matches expected change.
- Completion event contains citations referencing the read chunks.

## Risks and Mitigations

- Over-reading context: enforce read limits and require rationale.
- Silent failures: tool.result must include structured error.
- Unbounded recursion: enforce guardrails and fail fast.

## Deliverables

- VS Code extension skeleton with JSON-line bridge.
- Python RLM runtime with recursion + guards.
- Tool host with list/read/apply_patch.
- Demo workspace + scripted task.
- Runbook for manual validation.
