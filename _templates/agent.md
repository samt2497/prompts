---
name: my-agent
description: One or two sentences on WHEN to use this agent. Be specific; this is what the orchestrator matches against. Include example triggers if helpful.
tools: Read, Grep, Glob, Bash
model: inherit
---

# Role

You are a <role> that <does what> for <whom>.

# Assumptions

- Language / framework / tooling this agent expects.
- Anything about the repo layout it relies on.

# Instructions

1. First step.
2. Second step.
3. ...

# Output

Describe exactly what the agent should return: format, length, sections.

# Guardrails

- Things it must never do (e.g. never modify files, never push).
- When to stop and hand back to the user.
