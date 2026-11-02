# claude-run

A Claude Code skill for running the `claude` CLI as a non-interactive worker
from another agent, a script, or a CI job.

`SKILL.md` covers:

- `claude -p` and how the working directory is chosen;
- prompt files, stdin (and its 10 MB cap), and prompts after list flags;
- system prompts, `CLAUDE.md`, skills, and bare mode;
- permission modes, tool lists, restricted mode, and turn and budget caps;
- text, JSON, stream-JSON, and JSON Schema output;
- reviewing a piped diff with read-only tools;
- session IDs, resume and fork, and session persistence;
- background work, process ownership, and stopped or failed runs; and
- model aliases, pinned model IDs, and effort levels.

The skill was checked against Claude Code 2.1.316 and its documentation.

## Install

Put this directory in a Claude Code skills directory, such as
`~/.claude/skills/claude-run/` or `.claude/skills/claude-run/` in a project.
Keep the directory name `claude-run`. Claude Code must already be installed
and signed in; this skill does not install or authenticate it.

The CLI changes between releases. When a flag is uncertain, check:

```bash
claude --help
```

The license is Apache-2.0; see [LICENSE](LICENSE).
