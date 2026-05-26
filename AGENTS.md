# AGENTS.md

## Skill Maintenance

- Keep skills in `skills/<category>/<skill-name>/SKILL.md`.
- Do not set per-skill version fields in `SKILL.md`; rely on git/plugin marketplace versioning instead.
- When adding, deleting, moving, or renaming a skill, update `.claude-plugin/marketplace.json` in the same change so Claude Code and `npx skills add` discovery stay in sync.
- When changing the skill inventory or category structure, update `README.md` to match.

## Plugin Marketplace

- `.claude-plugin/marketplace.json` is the source of truth for the installable Claude Code plugin and `npx skills add` discovery.
- Use `strict: false` for the current layout, where the marketplace entry directly lists skill directories and no per-plugin `.claude-plugin/plugin.json` files are used.
- Keep plugin `skills` paths relative to the repository root and pointing at directories that contain a `SKILL.md` file.
- Omit explicit plugin `version` fields
