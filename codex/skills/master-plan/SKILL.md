---
name: master-plan
description: Use when the user wants to continue a multi-session goal tracked in a master plan document (a cross-runtime format), says "master plan", or references a master plan file path. Resumes an existing plan by assessing the gap, proposing the next step, gating execution on approval, and appending a session entry. Does not create new master plans in this runtime — those should be authored in Claude Code or drafted by hand.
metadata:
  short-description: Continue an existing master plan
---

# Master Plan (Codex adapter, v1)

Codex-native adapter for the cross-runtime master-plan workflow. Produces session entries in the same master-doc format a Claude session would; the master doc is portable between runtimes.

## What this skill does

Resumes an existing master plan: reads the doc, assesses the gap against the vision, proposes the highest-leverage next step, gates execution on user approval, executes, and appends a session entry.

## What this skill does NOT do (Codex v1 subsets)

Each subset is an explicit degradation under the master-plan principle *"Adapters may degrade, not forced to mimic"*:

- **No new-plan setup (Phase 1).** Creating a master plan from scratch requires a brainstorming loop that currently only exists in the Claude version. Start a new plan there, or draft the master doc by hand from the format declared in its own `Schemas` section (if present) or by copying the shape of an existing master doc.
- **No subagents.** Post-execution steps (consistency check, commit, bookkeeping) run inline in the main session.
- **No parallel tracks.** Session entries are always sequential (per Schema 2: a runtime without a parallel primitive writes sequential entries only; this is that declaration).
- **No automatic code review every 5 sessions.** The `Review` field in session entries is therefore omitted by policy rather than per-session.
- **No master-doc coherence cadence.** Same policy reason.
- **No convergence checklist.** If the user decides the plan is done, they mark the final session entry manually or switch to Claude for the full convergence flow.
- **No bridge skill.** Session context that needs to survive a reset lives in the master doc; anything that needed a bridge belongs in the bookkeeping entry.
- **No automated context-budget signal.** Codex has no standard file for used-context percentage. If the user wants budget awareness, ask them what their usage is before executing; otherwise proceed.

## Storage

- **Master docs** live at the location named in the project's `CLAUDE.md` (or `AGENTS.md`) under a `Master plans directory` entry, or `docs/master-plans/` if no such entry exists. This location is shared across runtimes — do not move it.
- **Session plans** (scratch written during Step 3) live at `~/.codex/plans/YYYY-MM-DD-topic.md`. Create the directory if missing. This is runtime-local scratch; other runtimes will not read it.

## Routing

- **Master-doc path provided** (e.g. the user says "continue master plan at `…md`") → go straight to Phase 2.
- **No path provided** → list master-doc filenames under the configured directory, ask which to resume. Do not offer to start a new one; that's out of scope.

## Phase 2: Execute

Announce at start: "Starting execution session. I'll assess the current state, propose the next step, align on a plan, then execute."

### Step 1 — Assess

1. Read the master doc in full, including every `Previous sessions` entry.
2. Assess the current state against the Vision, fresh each time. Do not carry forward a prioritized list from a prior session.
3. If the master doc contains a design-artifact section (any named section other than `Vision`, `Principles`, `Schemas`, `Starting state`, `Previous sessions` — e.g. "Target pipeline" or "Component architecture"), verify the current implementation matches it and flag structural drift.
4. Present the assessment as: what the vision describes / what you see now / the remaining gap.

### Step 2 — Propose

1. Propose the highest-leverage next step and its verification criteria.
2. Reason explicitly about leverage: which part of the vision this advances, whether it is cleanup or a real-gap closer, whether higher-leverage alternatives exist.
3. Check principle and design compliance: does the step bend any principle? Does it deviate from a design artifact? If yes, name the principle or artifact and get explicit user approval before proceeding — do not silently take the shortcut.
4. Do not attempt convergence detection in v1. If the user thinks the plan is complete, they will say so, and the skill hands off to them for the convergence steps.

### Step 3 — Plan (alignment gate)

This step replaces Claude's `EnterPlanMode` / `ExitPlanMode`. Codex has no plan-mode primitive, so alignment happens through a plain markdown file plus an explicit user-approval gate.

1. Create `~/.codex/plans/YYYY-MM-DD-topic.md` with these sections: **Scope**, **Approach**, **Verification criteria**, and the **Post-execution checklist** shown in Step 5 (copy it verbatim, replacing `<MASTER_DOC_PATH>` with the actual path).
2. Show the plan file to the user and ask for approval. Do not start editing other files until the user says yes.
3. If the user requests changes, edit the plan file and re-present.

### Step 4 — Execute

1. Do the work described in the plan file.
2. Verify against the plan's Verification criteria. If any criterion fails, fix before moving on.

### Step 5 — Post-execution

All steps run inline in the main Codex session (no subagents). Do not skip. Complete in order:

```
## Post-execution (do not skip)
After implementation and verification, complete ALL of these steps:

1. CONSISTENCY CHECK — inline:
   - Check documentation consistency (CLAUDE.md / AGENTS.md, README, inline comments).
   - Check pattern consistency across sessions (naming, structure, style).
   - Check for stale references from previous sessions.
   - Fix any issues found.
   - Record: what was checked and what was fixed (or "clean") — used in the session entry below.

2. COMMIT — stage all session changes and create a commit:
   - Run `git diff --stat` to review what changed.
   - Stage the relevant changes (implementation + consistency fixes from step 1).
   - Create a commit summarizing the session's work.
   - Record the commit hash(es) for the bookkeeping step.

3. BOOKKEEPING — edit the master doc at <MASTER_DOC_PATH> directly:
   - Read the master doc.
   - Scan the existing `## Previous sessions` list to find the next number N.
   - Append a single numbered entry in this exact format:

     N. *YYYY-MM-DD -- topic* -- One-line summary. Plan: `~/.codex/plans/YYYY-MM-DD-topic.md`. Commits: `abc1234`. Verified: [what was checked]. Consistency: [findings or "clean"].

     The `Review` field is omitted by policy (see "What this skill does NOT do"), not per session.

   - Do not modify Vision, Principles, Schemas, Starting state, or any prior session entry.
   - Write the file atomically (one write, not a sequence of partial edits).

4. HANDOFF — tell the user: "Master doc updated at <MASTER_DOC_PATH>. Start a fresh Codex session when ready, or continue in Claude Code."
```

## Schema conformance

This skill produces Schema-2 sequential session entries. Fields it emits:

| Field | Value |
|---|---|
| `date` | session date |
| `topic` | short phrase |
| `summary` | one-line description |
| `Plan` | `~/.codex/plans/YYYY-MM-DD-topic.md` (informational only; other runtimes cannot open it, but the entry is still valid per Schema 2) |
| `Commits` | hash list from step 2 |
| `Verified` | list from Verification criteria in the plan file |
| `Consistency` | findings from step 1 (or "clean") |
| `Review` | **omitted by policy** — see "What this skill does NOT do" |

## Key behaviors

- **No stale state.** Never carry forward a prioritized list; assess fresh each session.
- **No loose steps.** Every action the skill expects to happen is a named step above. If you catch yourself doing something that isn't in the checklist, stop and ask.
- **Domain-agnostic.** Works for code, docs, prompts, processes.
