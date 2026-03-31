# claude-master-plan

## Releasing changes

When modifying the skill, always:

1. **Bump version** in `.claude-plugin/plugin.json` before committing
2. **Commit and push** to master
3. **Refresh the marketplace cache** -- `claude plugin marketplace update claude-master-plan`
4. **Update the plugin** -- `claude plugin update -s user master-plan@claude-master-plan`
5. **Restart Claude Code** to apply

Step 3 is required because `plugin update` checks the cached marketplace, not the remote. Without it, the update command will report "already at latest."

## Design principle: no loose steps

Every action the skill expects to happen must be owned by a plan file. If a step exists only as "the orchestrator will do it" without being written into a plan's sequence, it will get skipped. When adding new flows or modifying existing ones:

- Ask: "who executes this step, and is it in their plan?"
- If the main conversation (orchestrator) executes it, it must be a named section in the session plan
- If a subagent executes it, it must be in the agent's prompt (which comes from a plan section)
- Post-agent steps (merge, combined verification, conflict resolution) are especially easy to leave unowned -- always check these

## D2 diagrams

When modifying `.d2` files, always re-render the corresponding `.svg` by running `d2 <source>.d2 <output>.svg`. Never commit a changed `.d2` without updating its `.svg`.
