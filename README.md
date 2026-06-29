# InstaCloud Skills

Agent skills (Markdown in the [Anthropic Skills format](https://docs.claude.com)) that teach AI
coding agents how to operate cloud resources through an **InstaCloud** project — branching,
deploying, the credential seam, governance, and running multiple agents in parallel.

## Layout

```
skills/
├── README.md            ← you are here
└── insta/
    ├── SKILL.md         ← entry skill: the model, setup, two non-negotiables, governance
    ├── workflow.md      ← branch → build → deploy → test → promote; parallel agents; gotchas
    └── cli-reference.md ← full `insta` command catalog, deploy, Dockerfiles, govern/observe
```

## Skill format

A skill is a `SKILL.md` with YAML frontmatter:

```markdown
---
name: <skill-name>
description: <one-line description the agent uses to decide whether to invoke this skill>
---

# <title>
<markdown body — instructions, examples, gotchas, references to the `insta` CLI>
```

## Authoring guidelines

- **Reference the `insta` CLI, not provider APIs.** Skills tell the agent which `insta` command to
  run; provider-specific knowledge lives behind the CLI and control plane.
- **One skill, one capability.** Split large topics by task.
- **Include failure modes.** A skill is most valuable when it teaches what goes wrong and how to recover.
- **Keep examples runnable.** Agents execute the code blocks.

## Install into a project

```bash
# (planned) insta skills pull — for now copy skills/insta/ into your agent's skills dir, e.g.
mkdir -p .claude/skills && cp -r skills/insta .claude/skills/insta
```

The `insta observe` credential-audit hook is installed automatically on `insta project create`/`link`
(see SKILL.md → Governance & audit).
