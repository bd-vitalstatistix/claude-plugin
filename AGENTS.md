# Agent Notes — Claude Plugin

## Version Bumping

Do **not** manually edit `.claude-plugin/plugin.json` to bump the version.

A GitHub Actions workflow (`.github/workflows/bump-version.yml`) auto-increments the patch version on every push to `main`. Just commit your skill changes; the bump happens automatically.

## Adding a New Skill

1. Create `skills/<skill-name>/SKILL.md` with YAML frontmatter:
   ```yaml
   ---
   description: One-line summary of what the skill does
   disable-model-invocation: false
   ---
   ```
2. Add the skill to the table in `README.md`
3. Commit — the version bump workflow handles the rest

## Validation

```bash
claude plugin validate .
```
