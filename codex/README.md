# Codex adapter

Codex-native adapter for the master-plan workflow. Produces session entries in the same format as the Claude version, so a single master doc can be continued on either runtime.

## What this is

A Codex skill that resumes an existing master plan: assesses the gap, proposes the next step, executes, and appends a session entry. It does **not** create new master plans — for the brainstorming phase use Claude Code, or draft the master doc by hand.

## Install

Codex auto-discovers skills under `~/.codex/skills/<name>/SKILL.md`. Symlink this directory into that location:

```bash
ln -s "$(pwd)/codex/skills/master-plan" ~/.codex/skills/master-plan
```

Or copy if you prefer a snapshot:

```bash
cp -R codex/skills/master-plan ~/.codex/skills/master-plan
```

Verify:

```bash
ls ~/.codex/skills/master-plan/SKILL.md
```

No plugin manifest, no marketplace step, no restart — Codex picks the skill up on the next session.

## Usage

Codex skills are triggered by natural-language intent, not slash commands. In a Codex session say something like:

- `continue master plan at /path/to/master.md`
- `resume master plan` (the skill will list candidates under the configured directory)

The skill lists existing master docs, proposes the next step, gates on your approval via a plan file at `~/.codex/plans/YYYY-MM-DD-topic.md`, and appends a session entry to the master doc when done.

The master doc location is read from a `Master plans directory` entry in the project's `CLAUDE.md`, the project's `AGENTS.md`, or the global `~/.codex/AGENTS.md`; it falls back to `docs/master-plans/` if present.

### First run — preflight

On the first invocation in a conversation, the skill runs a short preflight that checks two things:

1. **`~/.codex/plans/` is writable under your Codex sandbox.** Codex's default `workspace-write` sandbox mode blocks writes outside the active workspace, so this typically fails until you add `~/.codex/plans` to `writable_roots` in `~/.codex/config.toml`. The skill offers to auto-patch the config, show you the manual fix, or fall back to an in-workspace scratch directory.
2. **A master-plans directory is resolvable.** If you didn't pass an explicit path and no `Master plans directory` entry is configured, the skill shows a menu with dotfiles-style and in-repo options and offers to persist your choice into CLAUDE.md / AGENTS.md.

Preflight only runs once per conversation — once both checks pass, subsequent invocations skip it. The full check + menu spec lives in `codex/skills/master-plan/SKILL.md` under `## Preflight`.

## What's subset vs. the Claude version

The Codex v1 adapter intentionally ships a minimal subset. See `codex/skills/master-plan/SKILL.md` for the full list with rationales; the headline omissions are:

- No brainstorming to create a new master plan.
- No subagents — post-execution steps run inline.
- No parallel tracks — session entries are sequential-only.
- No automated code review or master-doc coherence cadence.
- No convergence checklist.
- No bridge skill.
- No automated context-budget signal.

Each subset is an explicit degradation under the cross-runtime "adapters may degrade" principle, not an oversight.

## Schema conformance

Session entries written by this skill conform to Schema 2 of any master doc that defines cross-runtime schemas. The `Review` field is omitted by policy (the skill does not run automated reviews), in line with Schema 2's rule that omissions must be a procedural declaration — not a per-session choice.

The `Plan` field always records the actual `~/.codex/plans/…md` path, even though subsequent Claude sessions cannot open that file. Schema 2 treats plan paths as informational-only.
