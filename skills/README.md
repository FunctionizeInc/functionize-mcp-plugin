# Adding a skill

A skill is a folder with one `SKILL.md` file:

```
skills/
  your-skill-name/
    SKILL.md
```

```markdown
---
name: your-skill-name
description: One or two sentences on what this does and when to trigger it.
             Be concrete about trigger phrases/situations, not just a topic label.
---

Instructions for Claude go here: what to do, in what order, what to check,
what "done" looks like.
```

The frontmatter `description` is what Claude reads to decide when to
trigger the skill, so it has to say plainly what the skill does and when to
use it. Reference material a skill needs (templates, longer examples) can
live alongside `SKILL.md` in the same folder.

Once installed, a skill is invoked as `/functionize-mcp:your-skill-name`
(the `functionize-mcp` prefix comes from this plugin's name in
`.claude-plugin/plugin.json`).

## Testing before you send one over

```bash
claude --plugin-dir /path/to/this/repo
/functionize-mcp:your-skill-name
```

## What's a good candidate

Anything that captures know-how built up working against the hosted MCP
tools: how to read a session's status correctly, common failure shapes and
what they actually mean, how to verify a run really passed rather than
just started, sequencing gotchas across the ten tools. If it's something
you'd want to tell a new teammate before they touch this, it's a good
candidate for a skill.
