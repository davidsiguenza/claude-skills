# Claude Code Skills — David Siguenza

Custom skills for [Claude Code](https://claude.ai/code) organized by domain.

## Structure

```
salesforce/
  b2b-commerce/
    sf-b2b-demo-builder/       ← Entry point / orchestrator
    sf-b2b-store-generator/    ← Creates Experience Site + WebStore
    sf-b2b-catalog-generator/  ← Generates product catalog CSV
```

## How to install a skill

Copy (or symlink) a skill folder into one of:
- `~/.claude/skills/<skill-name>/` — global for Claude Code
- `~/.cursor/skills/<skill-name>/` — global for Cursor
- `<project>/.claude/skills/<skill-name>/` — project-scoped

Each skill folder must contain a `SKILL.md` at its root.

## Salesforce B2B Commerce package

Three coordinated skills to build end-to-end B2B Commerce demos on SDO / sandbox orgs.

| Skill | Purpose |
|-------|---------|
| `sf-b2b-demo-builder` | Orchestrator — routes between store and catalog |
| `sf-b2b-store-generator` | Phases A–H: Experience Site, WebStore, pricebooks, buyer groups, branding |
| `sf-b2b-catalog-generator` | Steps 1–7: product research, images, CSV, SFDX metadata, variations |

Start with `sf-b2b-demo-builder` for full demos.
