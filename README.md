# Tenkara Automations Skillset

Skills for building and operating Tenkara automations on SuperAgent.

## Skills

- **[choose-execution-mode](./choose-execution-mode/SKILL.md)** — Decide whether a new Tenkara automation should run via API/connector, Browser Use (Container or BrowserBase host), or Computer Use. Run this before building any new automation.

## Using this skillset in SuperAgent

In SuperAgent → Settings → Skillsets, paste this repo's HTTPS URL into the "Add a skillset repository" field and click **Add**. SuperAgent reads `index.json` from the repo root and registers every listed skill.

## Adding a new skill

1. Create a new folder at the repo root, named exactly the same as the skill's `name` frontmatter field (lowercase, hyphens, no spaces).
2. Put a `SKILL.md` inside it with valid frontmatter (`name`, `description` required) and instructions.
3. Compute the SHA-256 of `SKILL.md`:
   ```
   shasum -a 256 my-skill/SKILL.md
   ```
4. Add an entry to `index.json` under `skills[]` with the digest.
5. Commit and push. SuperAgent picks up the new skill on next refresh.

## Spec

This skillset follows the [Agent Skills v0.2.0 spec](https://agentskills.io/specification). Schema URI: `https://schemas.agentskills.io/discovery/0.2.0/schema.json`.
