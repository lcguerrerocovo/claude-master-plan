---
name: master-plan
description: Use when the user has a large goal spanning multiple sessions, wants to track progress across /clear boundaries, or says "master plan". Also use when resuming work on an existing master plan document.
---

# Master Plan

## Arguments

- `/master-plan` -- no args: start a new goal (brainstorming) or list existing master plans to continue
- `/master-plan <path/to/master.md>` -- continuation: read the doc and proceed to the next execution session

## Storage

- Check `CLAUDE.md` for a **Master plans directory** entry -- use that path if present
- If none found, default to `docs/master-plans/` relative to the project root
- If the directory doesn't exist, create it on first use
- Master docs live here; session plans are created via plan mode and persist to `~/.claude/plans/`

## Routing

- **Path provided** -> go to Phase 2: Execute
- **No args** -> check for existing master plans in the directory
  - **Existing plans found** -> list them, ask which to continue or start new
  - **No existing plans** -> go to Phase 1: Setup

---

## Phase 1: Setup (New Goal)

Collaborative brainstorming to converge on a master doc. This is NOT the brainstorming skill -- it's a focused dialogue to define the vision.

1. **Ask questions to understand** what the user wants to achieve -- one at a time
2. **Push back firmly on vague vision statements** -- ask "what would someone see when this is done?"
3. **Suggest principles** based on the domain and the conversation so far
4. **Be flexible on principles** -- they can evolve across sessions
5. **Write the master doc** once aligned (use the template below)
6. **Suggest clearing context** -- the brainstorming conversation is no longer needed. Tell the user: "Master doc saved. Run `/clear` and then `/master-plan <path>` to start the first execution session with fresh context."

<HARD-GATE>
Do NOT write the master doc until you and the user are aligned on both the vision and principles. A vague vision derails everything.
</HARD-GATE>

### Master Doc Template

```markdown
# [Goal Name]

## Vision
[What "done" looks like -- qualitative, not a checklist. Describe the end state
someone would see when looking at the result.]

## Principles
### 1. [Principle name]
[How work should be done -- style, patterns, constraints.
These are stable across sessions but can evolve.]

### 2. ...

## Starting state
[Current state entering this effort -- what works, what doesn't.
Static snapshot -- not a task list, no strikethrough tracking.]

## Previous sessions
1. *YYYY-MM-DD -- topic* -- One-line summary. Plan: `path/to/plan.md`. Commits: `abc1234`, `def5678`. Verified: [what was checked]. Consistency: [findings or "clean"]. Review: [findings or "clean"].
```

---

## Context Budget (optional)

The skill can monitor context window usage to avoid auto-compaction mid-work. This requires the user's status line command to write context stats to `/tmp/claude-context-window.json`. If the file doesn't exist, the skill falls back to asking the user.

**How to check:** Read `/tmp/claude-context-window.json`. If the file exists, extract `used_percentage`. If it doesn't exist, ask the user: "What's your context usage at? (check your status bar)"

**Thresholds:**
- **< 60%** -- safe to continue
- **60–75%** -- warn: "Context is at N%. This execution cycle should be the last before /clear."
- **> 75%** -- recommend checkpoint: "Context is at N%. Recommend saving a checkpoint and doing /clear before starting more work."

**When to check:**
- Before Step 4 (Execute) -- if above 75%, recommend /clear instead of executing
- At the HANDOFF decision point -- inform the "continue vs /clear" recommendation

---

## Phase 2: Execute (Each Session)

This is the core loop. Every continuation session follows these steps.

**Announce at start:** "Starting execution session. I'll assess the current state, check convergence, and propose the next step."

### Step 1: Assess

1. Read master doc and previous session summaries
2. Assess current state against the vision -- fresh each time, no stale list
3. **Check design artifacts:** If the master doc contains a design specification section (e.g. "Target pipeline", "Component architecture"), verify the current implementation matches it. Flag any structural drift -- places where the implementation works but doesn't match the agreed design.
4. Present the assessment: "Here's what the vision describes. Here's what I see now. The remaining gap is: ___"

### Step 2: Check Convergence

If the remaining gap is small or remaining work is diminishing returns:
- Proactively recommend stopping
- If the user agrees, run the **convergence checklist** (replaces the normal post-execution checklist for this session):

  1. **CODE REVIEW (if needed)** -- check the "Previous sessions" entries
     to find the last session that included a code review. If there are
     unreviewed sessions since then, dispatch a subagent to review the
     accumulated diff from those sessions. Fix any issues found and
     commit the fixes. Skip if the previous session already ran a review.
  2. **COMMIT HISTORY REWRITE** -- reorganize all commits across all
     sessions into a logical narrative that tells the story of what
     was built and why. Do not squash — restructure into coherent,
     well-ordered steps. Must run before bookkeeping so the final
     session entry records post-rewrite hashes.
  3. **BOOKKEEPING** -- dispatch a subagent to update the master doc:
     mark the master plan as **closed/complete** in the session entry.
     Use the post-rewrite commit hashes from step 2. If code review
     (step 1) produced fixes, include those commit hashes too.
  4. **MASTER DOC COHERENCE** -- same as normal post-execution step 5
     (runs unconditionally here, not just every 5 sessions).
  5. **HANDOFF** -- tell user: "Master plan complete."

Do NOT push to continue when the vision is substantially met.

### Step 3: Propose Next Step

1. Propose the highest-leverage next step with verification criteria
2. **Explicitly reason about leverage:**
   - State which part of the vision this step advances
   - Flag if proposed work is cleanup/polish vs. closing a real gap
   - Prefer steps that move multiple aspects of the vision forward
   - Push back on incremental work when higher-leverage options exist
3. **Check principle & design compliance:**
   - Does this step maintain alignment with every principle? If it takes a shortcut that bends a principle, name the principle and get explicit user approval before proceeding.
   - If a design specification exists in the master doc, does this step build on it or deviate from it? Deviations require updating the design first.
   - Is the step *structurally* convergent (moving the architecture toward the design) or only *symptomatically* convergent (improving metrics while leaving the structure wrong)?
4. **Consider whether a design artifact is needed:**
   - Are implementation sessions building on structural assumptions that haven't been written down?
   - Have principles been describing constraints that the implementation is working around rather than satisfying?
   - If yes to either: propose a design session that produces a binding artifact in the master doc (a named section like "Target pipeline" or "Component architecture"). This artifact is checked in subsequent sessions the same way principles are. Updates require explicit user approval.
5. **Check for parallel candidates** -- after forming the proposal, check whether it decomposes into independent sub-steps:
   - **Independence test:** "Could two people do these without talking to each other?" -- no shared files, no ordering dependency, no shared state.
   - If yes, present named tracks (e.g. "Track A: …", "Track B: …"), state which are independent, and offer: "These tracks are independent -- run in parallel or sequential?"
   - **User chooses.** The skill never auto-parallelizes. If the user chooses sequential, or the step is inherently sequential, proceed with the normal Step 4 flow.
6. Discuss and align with the user before executing

### Step 4: Execute

**Do NOT invoke `superpowers:brainstorming` or `superpowers:writing-plans` during this step.** Master-plan owns its own planning process. Always use `EnterPlanMode` directly.

0. **Context check** -- before starting execution, check the context budget (see "Context Budget" section). If above 75%, recommend the user do `/clear` and restart with `/master-plan <path>` instead of executing. If 60–75%, warn that this should be the last cycle. Proceed only if the user confirms.

1. Use `EnterPlanMode` to create the session plan with these sections:
   - **Scope:** what this session will accomplish
   - **Approach:** how the work will be done
   - **Verification criteria:** how to confirm the work is correct
   - **Post-execution checklist** -- copy this verbatim into every session plan, replacing `<MASTER_DOC_PATH>` with the actual path:
     ```
     ## Post-execution (do not skip)
     After implementation and verification, complete ALL of these steps:

     1. CONSISTENCY CHECK -- dispatch a subagent to:
        - Check documentation consistency (CLAUDE.md, README, inline comments)
        - Check pattern consistency across sessions (naming, structure, style)
        - Check for stale references from previous sessions
        - Fix any issues found
        - Report: list what was checked and what was fixed (or "clean")

     2. COMMIT -- stage all session changes and create a commit:
        - Run `git diff --stat` to review what changed
        - Stage all relevant changes (implementation + consistency fixes
          from step 1)
        - Create a commit with a message summarizing the session's work
        - Record the commit hash(es) for the bookkeeping step

     3. CODE REVIEW (every 5 sessions) -- if this is session 5, 10, 15,
        etc., dispatch a subagent to:
        - Review the accumulated diff since the last code review
          (all commits from the last 5 sessions)
        - Check for correctness, safety, adherence to principles
        - Fix any issues found and commit the fixes
        - Report: list findings and fixes (or "clean")

     4. BOOKKEEPING -- dispatch a subagent to:
        - Find the session plan: run `ls -t ~/.claude/plans/ | head -5`,
          confirm with the user
        - Read the master doc at <MASTER_DOC_PATH> and the session plan
        - For sequential sessions: use the commit hash(es) from steps 2-3
        - For parallel sessions: the orchestrator must pass the agent
          commit hashes (collected during merge) into the subagent prompt,
          plus any post-merge commits from steps 2-3
        - Collect outcomes from steps 1 and 3 above
        - Add session to "Previous sessions" with: date, topic, summary,
          plan path (actual file, never "inline"), commits, what was
          verified, consistency findings, review findings
        - Propose vision/principle updates if warranted (present for approval)
        - For parallel sessions, use this bookkeeping format instead:
          N. *YYYY-MM-DD -- topic* -- Summary. Plan: `path`.
             - Track A: [summary]. Commits: `abc`. Verification: pass.
             - Track B: [summary]. Commits: `def`. Verification: pass.
          Verified: [combined]. Consistency: [findings]. Review: [findings].

     5. MASTER DOC COHERENCE (every 5 sessions) -- if this is session 5,
        10, 15, etc., dispatch a subagent to review the master doc for:
        - Resolved design questions that should be collapsed or archived
        - Sections that contradict each other or have drifted from the
          vision
        - Design artifacts that no longer reflect reality
        - Do NOT merge or remove session entries -- each session stays
          as its own record
        Present proposed edits for user approval before applying.

     6. HANDOFF -- tell user: "Master doc updated. Run /clear and then
        /master-plan <MASTER_DOC_PATH> to start the next session."
        If context budget is available and above 60%, also mention:
        "Context is at N% -- good time to /clear."
     ```
2. Execute the work
3. Verify against the criteria from the plan

#### When user chooses parallel execution

If the user chose parallel execution in Step 3, replace the normal execute flow above with this:

1. **Enter plan mode once** with an orchestrator plan containing these sections:
   - **Scope:** what this session will accomplish (same as normal)
   - **Tracks:** one subsection per track -- each with scope, approach, and verification criteria
   - **Merge sequence:** the order tracks merge in (plan order), conflict resolution policy, combined verification step
   - **Post-execution checklist:** same as normal (copy verbatim from template above)

2. **Dispatch all tracks as parallel Agent calls in a single message:**
   ```
   Agent(
     description: "Track A: ...",
     prompt: "<track section from orchestrator plan>\n\nMaster doc: <path>\n\nVerify your work against the track criteria. Commit passing work.",
     isolation: "worktree",
     model: "opus",
     mode: "bypassPermissions"
   )
   ```
   Agents do NOT get separate plan files -- they receive their track section from the orchestrator plan.

3. **Collect results** from each agent: what was done, verification pass/fail, branch name, commits.

4. **Merge sequence:**
   - Passing tracks merge in plan order
   - If a merge produces conflicts, present them to the user for resolution
   - Failed tracks are reported but not merged
   - After all passing tracks are merged, run combined verification on the merged result
   - **If combined verification fails:** present the failures to the user with context on which tracks likely caused the issue. Offer two options: (a) revert the merges and re-run failing tracks with fixes, or (b) fix forward in the current branch. User chooses.

Session plans are lightweight. If a session's scope is genuinely complex, suggest using the writing-plans skill as an exception.

### Step 5: Post-Execution

<HARD-GATE>
Do NOT skip this step. Do NOT suggest `/clear` or end the session before completing ALL items in the post-execution checklist from the session plan.
</HARD-GATE>

Follow the **Post-execution checklist** in your session plan. Consistency checks run every session. Code review and master doc coherence run every 5 sessions. None are gated on user approval. Use subagents for all steps to keep the main context clean.

---

## Key Behaviors

- **No stale state:** Never carry forward a prioritized list. Assess fresh each session from the master doc.
- **Domain-agnostic:** Works for code, docs, prompts, processes. Use "assess current state" not "read codebase."

## Red Flags -- STOP and Reassess

| Thought | Reality |
|---------|---------|
| "Let me just do this small cleanup" | Is it high-leverage? If not, propose something better. |
| "The list from last session said..." | There is no list. Assess fresh. |
| "We should keep going" | Check the gap. If it's small, recommend stopping. |
| "This principle needs work" | Principles evolve, but the vision is the north star. |
| "Let me plan all remaining sessions" | One session at a time. Fresh assessment each time. |
| "Results improved, so the approach is working" | Metrics can improve despite wrong architecture. Check structural convergence, not just output quality. |
| "We designed this but haven't built it yet -- let's do something else first" | If a design artifact exists and implementation doesn't match it, that's the highest-leverage gap. Don't build more on a wrong foundation. |
| "This shortcut is fine, we'll fix the structure later" | Later never comes. Name the principle it bends and get explicit approval. |
