# bug-fixing-agent

# Bug-Fixing Agent — Build Plan (v1)

> Hand this to Claude Code in **Plan Mode** first. Do not start writing code until the plan
> is reviewed and the scope boundaries below are respected.

## Goal

Build a **single-agent, tool-calling bug-fixing loop from the raw Anthropic API** to learn
agent internals: tool dispatch, context management, and a self-correcting THINK → ACT →
OBSERVE cycle grounded in real test execution.

The system takes a bug (a GitHub issue), lets an agent navigate a codebase, propose a patch,
verify it against the test suite, and open a **pull request** (never a merge) with an
execution-grounded confidence score. The PR is the human-in-the-loop gate.

## Non-negotiables (read before building)

1. **Build the agent loop on the raw Anthropic Messages API.** Define our own tools, our own
   dispatch, our own context management. **Do NOT shell out to Claude Code or any coding-agent
   CLI for the diagnosis/patch step** — the whole point is to build the loop ourselves. If the
   output resembles Claude Code, that means it was built correctly.
2. **No autonomous commits/merges.** Output stops at an opened PR for human review.
3. **The confidence score must be execution-grounded**, not model self-reported. Derive it from
   signals like: patch applied cleanly, existing tests still pass, target test went red→green,
   linter passed, number of files touched. A vibes number from the model is not acceptable.
4. **Retrieval is a tool the agent calls, not a preprocessing step.** Hybrid search lives
   *behind* a `search_codebase` tool the agent can call repeatedly with its own queries — not a
   one-shot context stuff before the loop starts.

## Out of scope for v1 (explicitly do not build these yet)

- Multi-agent orchestration / routing / specialized subagents. (v2, only after a single-agent
  baseline exists to measure against.)
- Ingestion adapters beyond GitHub issues (no Jira, no Excel).
- A natural-language "wiki" / per-file summary index. Start with `search_codebase` + `read_file`
  and only add a repo map if measurement shows it's needed.
- SWE-bench harness integration. (Later stretch; use own repo + hand-curated set first.)

## Tech stack

- **Language:** Python 3.11+
- **Model API:** Anthropic Messages API (tool use). Use a current model string — confirm the
  latest before building rather than hardcoding from memory.
- **Retrieval:** hybrid dense + BM25 with RRF fusion. Weaviate is fine (native hybrid, already
  familiar) but keep it behind the tool interface so it's swappable.
- **Sandbox:** run the target repo's tests in an isolated environment (venv or container per run;
  container preferred for isolation). Assume patches may be wrong — never run against the real repo.
- **GitHub:** REST API for fetching issues and opening PRs on a fix branch.

## Architecture — the agent loop

```
fetch issue
  → build initial query from issue title/body
  → AGENT LOOP (THINK → ACT → OBSERVE):
        tools available to the agent:
          - search_codebase(query)   # hybrid RRF retrieval, callable repeatedly
          - read_file(path, range)   # read surrounding code / context
          - find_references(symbol)  # optional, if cheap to add
          - apply_patch(diff)        # stage a patch in the sandbox
          - run_tests(target?)       # execute suite in sandbox, return pass/fail + output
      loop until: agent signals done, OR max-iteration budget hit
      on test failure → feed result back into OBSERVE so the agent self-corrects
  → assemble execution-grounded confidence score
  → open PR on fix/<issue-id> branch with patch + score + reasoning summary
  → STOP (human reviews PR)
```

Key design points to get right:
- **Context management:** decide what stays in the running message history vs what gets
  summarized/dropped as the loop grows. This is a core learning objective — do it deliberately.
- **Tool-result formatting:** test output and file contents can be huge; truncate/summarize
  before feeding back or the context blows up.
- **Iteration budget + termination:** hard cap on loop iterations and tool calls to avoid
  runaway cost. Log why the loop stopped.

## Build phases (milestones)

**Phase 0 — one bug, end to end, no eval.**
Wire the full loop for a single hand-picked bug from your own repo (the YouTube extension's
git history: a bug commit + its fix). Success = the agent produces a sensible patch and the
verification step runs. No metrics yet. Goal is to feel the loop working.

**Phase 1 — the tools, done properly.**
Flesh out `search_codebase` (hybrid RRF) and the sandbox `run_tests`. Confirm retrieval-as-a-tool
works: the agent calls search with its own queries mid-loop. Add the execution-grounded
confidence score.

**Phase 2 — hand-curated eval set.**
15–20 bugs from one well-chosen open-source repo with clean issue → merged-PR → test-change
linkage. Define a tight bug taxonomy for labeling. Measure fix rate (target test red→green,
existing tests still green). Report one honest number.

**Phase 3 (stretch) — SWE-bench Lite subset.**
Only after Phases 0–2 are solid. Use SWE-bench's existing harness — do not rebuild a sandboxed
runner. Prefer recent issues to reduce memorization inflation, or note the caveat explicitly.

## Data & eval plan

- Start: own repo git history (cheapest end-to-end signal).
- Then: hand-curated 15–20 issue set, fully understood, from one external repo.
- Stretch: SWE-bench Lite subset for external credibility.
- Metric for v1: **fix rate** (did the target test go red→green without breaking existing tests).
  Routing accuracy is a v2 concern (there's no router in v1).

## Build log convention

Keep a `BUILD_LOG.md` updated as you go. For each meaningful decision or failure, jot:
what broke, the hypothesis, what fixed it, and any tradeoff. This is both interview material
and progress tracking. Example entry shape:
```
## <date> — <short title>
- Symptom:
- Cause:
- Fix / decision:
- Tradeoff or open question:
```

## Definition of done for v1

- [ ] Agent loop runs on the raw Anthropic API with our own tool definitions and dispatch.
- [ ] `search_codebase` (hybrid RRF) and `read_file` callable by the agent mid-loop.
- [ ] Sandbox `run_tests` executes and feeds failures back into the loop for self-correction.
- [ ] Confidence score derived from execution signals, not model self-report.
- [ ] Produces an opened PR (fix branch) — no autonomous merge.
- [ ] Fix rate measured on the hand-curated 15–20 bug set, reported as one honest number.
- [ ] `BUILD_LOG.md` maintained throughout.

## Open questions to resolve during Plan Mode (don't guess — decide explicitly)

- Container vs venv for the sandbox given the target repo's dependencies?
- Iteration/cost budget per bug?
- Chunking strategy for indexing code for retrieval (enrich chunks with natural-language
  context so NL ticket queries retrieve code — the semantic-gap problem from AIRA applies here)?