# claude-master-plan

## Releasing changes

When modifying the skill, always:

1. **Bump version** in `.claude-plugin/plugin.json` before committing
2. **Commit and push** to master
3. **Nuke the plugin cache** -- `rm -rf ~/.claude/plugins/cache/claude-master-plan/`
4. **Refresh the marketplace cache** -- `claude plugin marketplace update claude-master-plan`
5. **Reinstall the plugin** -- `claude plugin uninstall -s user master-plan@claude-master-plan && claude plugin install -s user master-plan@claude-master-plan`
6. **Restart Claude Code** to apply
7. **Update snipe** -- `snipe update` to pull the latest commit into `~/.snipe/git/`

Steps 3-5 are needed because `plugin update` often reports "already at latest" without actually refreshing the cache directory. Nuking the cache and reinstalling forces a clean pull.

For snipe, the package is cloned from the git source on install and updated via `snipe update`.

## Versioning

Semantic versioning for `.claude-plugin/plugin.json`:

- **MAJOR** (X.0.0): breaking changes to master doc format or session plan structure (would break existing master plans)
- **MINOR** (0.X.0): new features or flow changes (new steps, new execution paths)
- **PATCH** (0.0.X): bug fixes, wording clarifications, diagram updates

## Design principle: no loose steps

Every action the skill expects to happen must be owned by a plan file. If a step exists only as "the orchestrator will do it" without being written into a plan's sequence, it will get skipped. When adding new flows or modifying existing ones:

- Ask: "who executes this step, and is it in their plan?"
- If the main conversation (orchestrator) executes it, it must be a named section in the session plan
- If a subagent executes it, it must be in the agent's prompt (which comes from a plan section)
- Post-agent steps (merge, combined verification, conflict resolution) are especially easy to leave unowned -- always check these

## D2 diagrams

- When modifying the flow in `SKILL.md`, always update `docs/master-plan-flow.d2` to match. The diagram must stay aligned with the skill definition.
- Keep the diagram vertical (`direction: down`) for readability in GitHub.
- If the diagram gets complex, simplify — but preserve the essential flow and key steps so a reader can understand what the master plan does at a glance.
- When modifying `.d2` files, always re-render the corresponding `.svg` by running `d2 <source>.d2 <output>.svg`. Never commit a changed `.d2` without updating its `.svg`.
