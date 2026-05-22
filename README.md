# pilotso11-skills

Personal collection of [Claude Code](https://www.anthropic.com/claude-code)
skills authored by [@pilotso11](https://github.com/pilotso11), packaged as a
single installable plugin.

## Installing in Claude Code

### One-line install (recommended)

From within Claude Code, run the slash command:

```
/plugin install pilotso11-skills@https://github.com/pilotso11/pilotso11-skills
```

Claude Code will fetch the latest `main`, register the plugin, and load every
skill under `skills/`.

### Local development install

If you want to hack on a skill locally, clone the repo and launch Claude with
`--plugin-dir` pointing at the checkout:

```bash
git clone https://github.com/pilotso11/pilotso11-skills.git
cd pilotso11-skills
claude --plugin-dir .
```

Claude will load the plugin for that session only, picking up local edits to
any `SKILL.md` without reinstalling.

## Available skills

| Skill | Trigger | What it does |
|---|---|---|
| `hello` | User greets Claude | Confirms the plugin is loaded and active (smoke test) |

## Publishing a new skill

1. **Create the skill directory:**
   ```bash
   mkdir -p skills/<skill-name>
   ```
2. **Write `skills/<skill-name>/SKILL.md`** with YAML frontmatter. `name` and
   `description` are required; the description is what Claude uses to decide
   when to invoke the skill, so be specific about triggering conditions.
   ```markdown
   ---
   name: <skill-name>
   description: Use when ... (describe the triggering situation precisely)
   ---

   # Skill body

   Instructions for Claude when this skill is invoked.
   ```
3. **Validate locally:**
   ```bash
   claude plugin validate .
   ```
   Resolve any errors before opening a PR. Warnings are tolerated but worth
   addressing.
4. **Add your skill to the table** in this README under *Available skills*.
5. **Open a pull request.** CI will run the same `claude plugin validate .`
   check; it must pass before merge.
6. **Merge.** Once on `main`, anyone who installed the plugin will pick up
   the new skill on their next `claude plugin update`.

## CI

Every push to `main` and every pull request runs `claude plugin validate .`
inside a `node:lts-slim` container with the official `@anthropic-ai/claude-code`
CLI installed. The job fails if `.claude-plugin/plugin.json` is missing or
malformed, or if any `skills/*/SKILL.md` has invalid structure.

Branch protection on `main` requires this check to pass before merge.

## License

[MIT](LICENSE)
