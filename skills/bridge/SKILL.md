---
name: bridge
description: Save context before /clear and restore it in the next session. Use when the user says "bridge", wants to carry context across a /clear, or needs to checkpoint a session.
---

# Bridge

Carry session context across `/clear` boundaries. Saves key context to a memory file, then restores and cleans up in the next session.

## Arguments

- `/bridge` -- no args: check for an existing bridge file. If found, restore it. If not found, create one.
- `/bridge save` -- explicitly save a bridge (create/overwrite)
- `/bridge restore` -- explicitly restore and delete a bridge

## Storage

The bridge file lives in the project's auto-memory directory alongside other memory files. It uses the standard memory file format with `type: bridge`.

- **File:** `bridge.md` in the project's auto-memory directory
- **Index:** one entry in `MEMORY.md`
- **Only one bridge exists at a time.** Saving overwrites any existing bridge.

## Save Flow

When the user invokes `/bridge` or `/bridge save`:

1. **Check context budget** -- read `/tmp/claude-context-window.json` if available. Include the context usage percentage in the bridge file for reference.

2. **Gather context** by reviewing the current conversation. Capture:
   - **What was just done** -- summary of work completed this session
   - **Key decisions** -- choices made and why (especially non-obvious ones)
   - **Gotchas and discoveries** -- things learned that aren't in the code or docs
   - **Current understanding of the gap** -- if working on a master plan, what remains
   - **What was about to happen next** -- if the session was interrupted mid-flow
   - **Active master plan** -- if a master plan is in progress, record its path

3. **Write the bridge file** to the auto-memory directory:

   ```markdown
   ---
   name: Session bridge
   description: Context checkpoint from previous session -- read on next session start, then delete
   type: bridge
   created: YYYY-MM-DD HH:MM
   context_usage: N%
   master_plan: <path or "none">
   ---

   ## What was done
   [Summary of work completed]

   ## Key decisions
   [Choices made and rationale]

   ## Gotchas and discoveries
   [Non-obvious things learned]

   ## Current gap
   [What remains relative to the goal]

   ## Next steps
   [What was about to happen or should happen next]
   ```

4. **Index it** -- add or update the entry in `MEMORY.md`:
   `- [Session bridge](bridge.md) -- context checkpoint, consume on next session`

5. **Tell the user:** "Bridge saved. Run `/clear` and continue in a fresh session. Start with `/bridge` to restore context."

## Restore Flow

When the user invokes `/bridge` or `/bridge restore` and a bridge file exists:

1. **Read the bridge file** from the auto-memory directory.

2. **Present the context** to the user:
   ```
   Restoring bridge from [date]:
   - Did: [summary]
   - Key decisions: [list]
   - Gotchas: [list]
   - Gap: [summary]
   - Next: [what was planned]
   ```
   If a master plan path is recorded, mention it: "This was mid-master-plan: `<path>`"

3. **Delete the bridge file** and remove its entry from `MEMORY.md`.

4. **Ask what to do next:**
   - If a master plan path was in the bridge: "Continue with `/master-plan <path>`?"
   - Otherwise: "What would you like to work on?"

## No-args Routing

When invoked as just `/bridge`:
- **Bridge file exists** -> restore flow
- **No bridge file** -> save flow

## Key Behaviors

- **One bridge at a time.** Saving always overwrites. No accumulation.
- **Bridges are ephemeral.** They exist only to cross one /clear boundary. Restore always deletes.
- **Don't over-capture.** Only save what would be lost on /clear and isn't already in files (master doc, code, commits). If everything important is already persisted, say so: "Nothing to bridge -- all context is in the master doc and commits."
- **Master-plan aware but not master-plan specific.** Works for any session, but knows to record and suggest the master plan path when relevant.
