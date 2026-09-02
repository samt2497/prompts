# Skills

On-demand instruction packs. Each skill is a folder with a `SKILL.md` and optional supporting files (references, scripts, examples). Format follows the Agent Skills spec used by Claude Code (`.claude/skills/<name>/SKILL.md`).

Install one: `cp -r skills/<purpose>/<name> .claude/skills/`

## Index

| Purpose | Skill | When it loads |
| ------- | ----- | ------------- |
| `ops/`  | [durable-batch-execution](ops/durable-batch-execution/SKILL.md) | A long batch job must resume cleanly and prove every item landed, instead of an agent reporting progress from memory. |

Template: [`../_templates/skill/SKILL.md`](../_templates/skill/SKILL.md)
