# claude-master-plan

## Releasing changes

When modifying the skill, always:

1. **Bump version** in `.claude-plugin/plugin.json` before committing
2. **Commit and push** to master
3. **Refresh the marketplace cache** — `claude plugin marketplace update claude-master-plan`
4. **Update the plugin** — `claude plugin update -s user master-plan@claude-master-plan`
5. **Restart Claude Code** to apply

Step 3 is required because `plugin update` checks the cached marketplace, not the remote. Without it, the update command will report "already at latest."
