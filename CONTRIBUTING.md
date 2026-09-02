# Contributing

Pull requests welcome. The bar is: would someone else actually reuse this?

## Conventions

- **One thing per file.** An agent, a skill, a workflow, or a prompt. Not a bundle.
- **Kebab-case names.** `review-pr.md`, not `Review PR.md`.
- **Frontmatter first.** Every entry starts with the frontmatter from its template. `description` must say *when* to use it, not just what it is.
- **State assumptions.** If it expects a language, framework, tool, or file layout, say so near the top.
- **No secrets, no client data.** Scrub names, keys, hostnames, and anything proprietary before committing.
- **Markdown only** unless the entry genuinely needs a script. If it does, keep it small and comment it.

## Where things go

| You have...                                              | Put it in     |
| -------------------------------------------------------- | ------------- |
| A role with tools and instructions                       | `agents/`     |
| Instructions that load on demand, maybe with helper files| `skills/`     |
| An ordered set of steps that uses several of the above   | `workflows/`  |
| A single prompt or template                              | `prompts/`    |

Then pick the purpose folder. If nothing fits, propose a new one in the PR and explain why.

## Checklist before opening a PR

- [ ] Copied from the right template in `_templates/`
- [ ] Frontmatter filled in, `description` is specific
- [ ] Tested at least once with a real model and it did what the description says
- [ ] Added to the index in the type folder's `README.md`
- [ ] No secrets or private details
