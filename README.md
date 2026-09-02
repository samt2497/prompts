# public-prompts

A public collection of prompts, agents, skills, and workflows I use day to day with AI coding tools. Everything here is meant to be copied, adapted, and reused.

Maintained by Simon Mora ([@samt2497](https://github.com/samt2497)).

## Layout

Top level is **what kind of thing it is**. Inside each, folders are **what it's for**.

```
.
├── agents/        Subagent definitions (a role + tools + instructions)
├── skills/        Reusable, on-demand instruction packs (SKILL.md format)
├── workflows/     Multi-step procedures that chain prompts, agents, or skills
├── prompts/       Standalone prompts: system prompts, one-shots, templates
└── _templates/    Starting points for adding new entries
```

Each of the four type folders is split by purpose:

| Purpose        | What goes here                                          |
| -------------- | ------------------------------------------------------- |
| `coding/`      | Writing, reviewing, refactoring, and debugging code     |
| `writing/`     | Docs, emails, posts, copy, editing                      |
| `research/`    | Investigation, summarisation, comparison, analysis      |
| `ops/`         | Infra, deploys, CI/CD, incident handling, migrations    |
| `productivity/`| Planning, note-taking, meetings, personal automation    |
| `media/`       | Image, video, and audio generation prompting            |

Add a new purpose folder when something clearly doesn't fit. Keep the set small.

## What each type means

**Agent.** A specialised assistant with a name, a description of when to use it, the tools it may call, and its instructions. In Claude Code these live in `.claude/agents/<name>.md`. See [`_templates/agent.md`](_templates/agent.md).

**Skill.** A packaged set of instructions loaded on demand when the task matches its description. Lives in a folder with a `SKILL.md` and optional supporting files. In Claude Code these go in `.claude/skills/<name>/SKILL.md`. See [`_templates/skill/SKILL.md`](_templates/skill/SKILL.md).

**Workflow.** A repeatable, multi-step process. Usually markdown a human or agent follows in order, sometimes with the agents and skills it depends on. See [`_templates/workflow.md`](_templates/workflow.md).

**Prompt.** Anything that stands alone: a system prompt, a one-shot request, a fill-in-the-blanks template. See [`_templates/prompt.md`](_templates/prompt.md).

## Using these

Most entries are plain markdown, so they work with any model or tool. For Claude Code specifically:

```bash
# Use an agent in one project
cp agents/coding/<name>.md .claude/agents/

# Use an agent everywhere
cp agents/coding/<name>.md ~/.claude/agents/

# Install a skill
cp -r skills/coding/<name> .claude/skills/
```

Prompts and workflows are copy-and-paste. Each file states at the top what it's for and any assumptions it makes.

## Adding an entry

1. Copy the matching template from `_templates/`.
2. Drop it in `<type>/<purpose>/`. Use a short kebab-case name.
3. Fill in the frontmatter. The `description` is what a tool uses to decide when to load it, so make it specific.
4. Add a line to the type folder's `README.md` index.

See [CONTRIBUTING.md](CONTRIBUTING.md) for conventions.

## License

[CC0 1.0](LICENSE), public domain. Use freely, no attribution required (though appreciated).
