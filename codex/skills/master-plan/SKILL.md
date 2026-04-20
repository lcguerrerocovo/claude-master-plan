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

- **Master docs** live at the location named in the project's `CLAUDE.md` (or `AGENTS.md`) under a `Master plans directory` entry, or `docs/master-plans/` if no such entry exists. This location is shared across runtimes — do not move it. Preflight Check 2 (below) resolves this on every invocation.
- **Session plans** (scratch written during Step 3) live at `~/.codex/plans/YYYY-MM-DD-topic.md`. Create the directory if missing. This is runtime-local scratch; other runtimes will not read it. Preflight Check 1 (below) probes this before Phase 2; if Codex's sandbox blocks the write, a fix menu is shown.

## Preflight

Run once per conversation before routing to Phase 2. If preflight already completed successfully earlier in this conversation, skip it. If any check fails, **stop** and present the menu below verbatim; wait for the user's reply before continuing.

### Check 1 — Session-plan storage writable

Probe:

```bash
mkdir -p ~/.codex/plans && touch ~/.codex/plans/.probe && rm ~/.codex/plans/.probe && echo OK
```

If the probe fails, Codex's `workspace-write` sandbox is blocking writes to `~/.codex/plans/`. Present this menu verbatim:

> The skill writes session plan files to `~/.codex/plans/`, but your Codex sandbox is blocking it. Pick a fix:
>
> 1. **Auto-patch `~/.codex/config.toml`** — add `~/.codex/plans` to `writable_roots` and `mkdir -p` the directory. You'll need to restart the Codex session for the config change to take effect, then re-invoke the skill.
> 2. **Show me the manual fix** — print the two-line change and exit without touching anything.
> 3. **Use in-workspace scratch for this session** — write the plan file to `./.codex-plans/` under the current workspace instead (adds `.codex-plans/` to `.gitignore` if missing).
>
> Reply with 1, 2, or 3.

Dispatch:

- **1 (auto-patch).** Resolve `~/.codex/config.toml`. If it's a symlink, edit the symlink *target* (the dotfiles source of truth — do not edit the symlinked destination). If a `[sandbox_workspace_write]` section exists, append `~/.codex/plans` to its `writable_roots` array if not already present; otherwise add the section:

  ```toml
  [sandbox_workspace_write]
  writable_roots = ["<absolute-home>/.codex/plans"]
  ```

  Then `mkdir -p ~/.codex/plans`. Tell the user: "Config patched. Restart Codex and re-invoke the skill — the sandbox change only takes effect on a fresh session." Exit.

- **2 (manual).** Print the `[sandbox_workspace_write]` block above plus the `mkdir -p ~/.codex/plans` command, note that Codex must be restarted after, and exit.

- **3 (in-workspace scratch).** Set the session-plan directory to `./.codex-plans/` for this run only. `mkdir -p ./.codex-plans`. If `.gitignore` exists and does not already list `.codex-plans/`, append it. Record the chosen path in the Step-3 plan file so the session entry's `Plan:` field (Schema 2) records the actual path used.

### Check 2 — Master plans directory resolvable

If the user's invocation already names an explicit master-doc path (e.g. "continue master plan at `/path/to/master.md`"), skip this check.

Otherwise, search for a `Master plans directory` entry in this order:

1. `./CLAUDE.md` in the workspace
2. `./AGENTS.md` in the workspace
3. `~/.codex/AGENTS.md` (or its symlink target in dotfiles)

As an implicit fallback, accept `docs/master-plans/` in the workspace if that directory already exists.

If nothing is usable, present this menu verbatim:

> No `Master plans directory` is configured and no `docs/master-plans/` exists in the current workspace. Where should I look for master docs?
>
> 1. `~/dotfiles/master-plans/` (runtime-neutral dotfiles path — recommended for cross-runtime use)
> 2. `~/dotfiles/claude/master-plans/` (legacy Claude-prefixed path — if your dotfiles still use this name)
> 3. `docs/master-plans/` in the current project (I'll create it)
> 4. Other — give me a path
> 5. Stop — I'll add a `Master plans directory` entry to CLAUDE.md / AGENTS.md myself first
>
> Reply with 1-5.

- **1-4.** Use the chosen path for this session. Then ask the persistence follow-up:

  > Persist this choice? I can add `- **Master plans directory:** <path>` to:
  >
  > a. `./CLAUDE.md`
  > b. `./AGENTS.md`
  > c. `~/.codex/AGENTS.md` (global for all Codex sessions)
  > d. Don't persist — this session only
  >
  > Reply with a-d.

  Apply the edit if the user picks a-c; skip if d.

- **5.** Print the entry shape (`- **Master plans directory:** /absolute/path/`) and exit.

### Check 3 — Git commit writable

Only relevant if the workspace is a git repo. Probe:

```bash
git rev-parse --is-inside-work-tree >/dev/null 2>&1 && \
  touch .git/.probe 2>/dev/null && rm .git/.probe && echo OK
```

- If `git rev-parse` fails, the workspace is not a git repo. Skip this check; Step 5 (Post-execution) step 2 (COMMIT) will record `Commits: none (not a git repo)` when it runs.
- If `git rev-parse` succeeds but the `touch` fails, Codex's sandbox is blocking writes to `.git/`. Even in `workspace-write` mode, Codex typically excludes VCS metadata by default — `writable_roots` entries may or may not override this depending on the Codex build. Present this menu verbatim:

> The skill's post-execution COMMIT step needs to write to `.git/`, but your Codex sandbox is blocking it. Pick a fix:
>
> 1. **Auto-patch `~/.codex/config.toml`** — add the workspace's absolute `.git/` path to `writable_roots`. You'll need to restart the Codex session for the change to take effect, then re-invoke the skill. Note: some Codex builds enforce the `.git/` exclusion independently of `writable_roots`; if restart doesn't clear the block, re-run and pick option 2 or 3.
> 2. **Show me the manual fix** — print the config snippet and a pointer to the Codex sandbox docs, then exit so you can investigate the right setting for your build (e.g. a `vcs_writes` flag, a different `sandbox_mode`, or a build-specific override).
> 3. **Skip commit for this session** — proceed to Phase 2. Step 5 (Post-execution) step 2 (COMMIT) will record `Commits: none (sandbox blocked .git writes; uncommitted changes at <paths>)` in the session entry per Schema 2 and Principle 3 ("adapters may degrade"). Changes stay uncommitted; commit manually from a shell afterwards if you want them recorded.
>
> Reply with 1, 2, or 3.

Dispatch:

- **1 (auto-patch).** Resolve `~/.codex/config.toml` (edit the symlink target if it's a symlink). If a `[sandbox_workspace_write]` section exists, append the workspace's absolute `.git/` path (e.g. `/abs/path/to/workspace/.git`) to its `writable_roots` array if not already present; otherwise add the section:

  ```toml
  [sandbox_workspace_write]
  writable_roots = ["<absolute-workspace>/.git"]
  ```

  Tell the user: "Config patched. Restart Codex and re-invoke the skill. If `.git/` writes are still blocked after restart, re-run and pick option 2 or 3 — some Codex builds enforce the exclusion independently of `writable_roots`." Exit.

- **2 (manual).** Print the `[sandbox_workspace_write]` snippet above, note that `writable_roots` may not be sufficient on all Codex builds, and suggest consulting `codex --help` and the Codex sandbox documentation for the authoritative setting. Exit.

- **3 (skip commit).** Record a session flag marking commits as skipped, and carry it into Phase 2. Step 5 (Post-execution) step 2 (COMMIT) honors the flag by running `git diff --stat` (read-only) to list uncommitted changes and recording `Commits: none (sandbox blocked .git writes; uncommitted changes at <paths>)` in the bookkeeping entry. Proceed to Routing.

### Preflight summary

If all three checks pass, proceed silently to Routing. If a check ended in "continue with a workaround" (Check 1 option 3's in-workspace scratch, or Check 3 option 3's skip-commit), carry that configuration into Phase 2 — the Step-3 plan file must record the actual session-plan directory used and Step 5 must honor the skip-commit flag so the session entry's `Plan:` and `Commits:` fields are accurate per Schema 2.

## Routing

Runs after Preflight.

- **Master-doc path provided** (e.g. the user says "continue master plan at `…md`") → go straight to Phase 2.
- **No path provided** → list master-doc filenames under the master-plans directory resolved in Preflight Check 2, ask which to resume. Do not offer to start a new one; that's out of scope.

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
   - If Preflight Check 3 set the skip-commit flag (sandbox blocked `.git/` writes), take the skip branch (this is an explicit Principle-3 degradation, not a silent skip):
     - Run `git diff --stat` (read-only) to list uncommitted changes.
     - Do NOT run `git add` or `git commit` — they will fail under the sandbox.
     - Record the bookkeeping entry's `Commits:` field as `none (sandbox blocked .git writes; uncommitted changes at <paths>)`, where `<paths>` is a compact representation of the `git diff --stat` result. Continue to step 3.
   - If the workspace is not a git repo: skip staging/committing. Record `Commits: none (not a git repo)` for the bookkeeping entry. Continue to step 3.
   - Otherwise (the normal path):
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
