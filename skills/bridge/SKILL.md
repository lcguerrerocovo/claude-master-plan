---
name: bridge
description: Save context before /clear and restore it in the next session. Use when the user says "bridge", wants to carry context across a /clear, or needs to checkpoint a session.
---

# Bridge

Carry session context across `/clear` boundaries. A subagent reads the raw
session transcript (`.jsonl`) and writes the bridge to a memory file, so
quality does not depend on what the main conversation still remembers. On
restore, the bridge is archived (not deleted) so a chain of work across
multiple clears stays traceable.

## Arguments

- No args: check for an existing active bridge. If found, restore it. If not found, save one.
- `save` -- explicitly save a bridge (create/overwrite the active one)
- `restore` -- explicitly restore and archive the active bridge

## Storage

- **Active bridge:** `bridge.md` in the project's auto-memory directory
- **Archive:** `<memory>/bridges/archive/YYYY-MM-DD-HHMM.md`
- **Index:** one entry in `MEMORY.md` for the active bridge only. Archives are not indexed.
- **One active bridge at a time.** Saving overwrites the active file; restoring moves it into the archive.

## Save Flow

When the user invokes the bridge skill to save:

### 1. Locate the current transcript

Find the raw session log for the current working directory. Session
transcripts are stored as `.jsonl` files in a harness-specific directory:

- **snipe:** `~/.snipe/sessions/<slug>--/<uuid>.jsonl` where `<slug>` encodes
  the working directory with `--` as path separators
- **Claude Code:** `~/.claude/projects/<slug>/*.jsonl` where `<slug>` encodes
  the working directory

Find the newest `.jsonl` matching the current working directory across
both locations.

### 2. Locate the predecessor (optional)

Check `<memory>/bridges/archive/` for the most recently created archive. If
one exists and was archived within the last 24 hours, treat it as a
predecessor so the new bridge can build on it rather than repeating it.

### 3. Capture context-budget reading (optional)

Read `/tmp/claude-context-window.json` (Claude Code) or check the snipe
status line if available. Pass this to the subagent so it lands in the
bridge frontmatter. If no programmatic source is available, skip the field.

### 4. Dispatch the bridge-writing subagent

Dispatch a subagent to write the bridge. The subagent runs in isolated
context, so this is cheap for the main conversation and safe even when
the main is near its context limit.

Prompt template (adapt values into each slot before dispatching):

```
You are writing a session bridge from the raw session transcript so we can
carry context across a /clear.

Inputs:
- Transcript (raw session .jsonl): <transcript_path>
- Output bridge file (overwrite if present): <memory_dir>/bridge.md
- Predecessor archive (may be absent): <predecessor_path or "none">
- Auto-memory directory: <memory_dir>
- Current date/time: <YYYY-MM-DD HH:MM>
- Context usage at save time: <N% or "unknown">
- Active master plan path: <path or "none">

Steps:
1. Read the transcript end-to-end. If it is long, read in chunks with
   Read offset/limit and keep a running summary. Work from the raw
   transcript, not any summary the orchestrator sends along.
2. If a predecessor is provided, read it. The new bridge should build on
   the predecessor: only capture what has changed or advanced since, and
   reference the predecessor path in the `predecessor` frontmatter field.
3. If everything worth carrying is already in files (master doc, code,
   commits), do NOT write the bridge. Instead, return a short message:
   "Nothing to bridge -- all context is in <files>."
4. Otherwise write <memory_dir>/bridge.md with this exact format:

   ---
   name: Session bridge
   description: Context checkpoint from previous session -- read on next session start, then archive
   type: bridge
   created: YYYY-MM-DD HH:MM
   context_usage: N%          # omit this line if unknown
   master_plan: <path or "none">
   predecessor: <archive path or "none">
   ---

   ## What was done
   [Summary of work completed this session]

   ## Key decisions
   [Choices made and why, especially non-obvious ones and rejected alternatives]

   ## Gotchas and discoveries
   [Non-obvious things learned that aren't in code or docs]

   ## Current gap
   [What remains relative to the goal, if working on a master plan]

   ## Next steps
   [What was about to happen or should happen next]

5. Report back in under 100 words: did you write the bridge, skip it, or
   build on a predecessor? Include the bridge path.

Capture the user's exact wording for pushback, rejected approaches, and
non-obvious decisions -- these degrade first under compaction and are the
highest-value things to preserve.
```

### 5. Update the MEMORY.md index

If the subagent wrote the bridge and there is no existing entry, add:

`- [Session bridge](bridge.md) -- context checkpoint, consume on next session`

If an entry already exists, leave it alone.

### 6. Tell the user

- If written: "Bridge saved (subagent wrote from raw transcript). Run `/clear` and start the next session by invoking the bridge skill to restore."
- If skipped: relay the subagent's "nothing to bridge" message.

## Restore Flow

When the user invokes the bridge skill to restore and an active bridge
exists:

1. **Read** `<memory>/bridge.md`.

2. **Present the context** to the user:
   ```
   Restoring bridge from [date]:
   - Did: [summary]
   - Key decisions: [list]
   - Gotchas: [list]
   - Gap: [summary]
   - Next: [what was planned]
   ```
   If `predecessor` is set, add: "Builds on archive `<path>`."
   If a master plan path is recorded, add: "This was mid-master-plan: `<path>`."

3. **Archive, don't delete.** Move `<memory>/bridge.md` to
   `<memory>/bridges/archive/YYYY-MM-DD-HHMM.md`. Use the timestamp from the
   bridge's `created` frontmatter when present; otherwise use the current
   time. Create the archive directory if it does not exist.

4. **Update the index.** Remove the `bridge.md` line from `MEMORY.md`.
   Archives are not indexed.

5. **Ask what to do next:**
   - If a master plan path was in the bridge: "Continue by invoking the master-plan skill with the path?"
   - Otherwise: "What would you like to work on?"

## No-args Routing

When invoked with no arguments:
- **Active bridge exists** -> restore flow
- **No active bridge** -> save flow

## Context-Budget Guidance

Because the save flow delegates to a subagent with isolated context, the
main conversation's context pressure does not directly compromise bridge
quality. Still:

- **Bridge earlier rather than later.** Target below 70% for save. Above
  85%, the main may already have been compacted once, which fuzzes the
  instructions it passes to the subagent even though the subagent still
  reads the raw transcript.
- **If the main notices mid-work that a compaction just happened**, prefer
  saving the bridge immediately -- the transcript is authoritative and the
  subagent will recover detail the main has lost.

## Key Behaviors

- **Subagent writes from raw transcript.** Bridge content is not
  double-summarized through the main conversation's compacted state.
- **One active bridge, many archives.** Archives preserve the chain of
  work across clears; the active bridge is what the next session restores.
- **Chain when related, start fresh when not.** If a recent archive
  exists, the subagent builds on it instead of repeating it. Unrelated
  prior work naturally ages out beyond 24 hours.
- **Don't over-capture.** The subagent skips writing when the master doc,
  code, and commits already cover everything worth carrying.
- **Master-plan aware but not master-plan specific.** Works for any
  session, but records and suggests the master plan path when relevant.
