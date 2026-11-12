# claude-run

A Claude Code skill for running the Claude CLI as a non-interactive worker
from another agent or automation harness.

`SKILL.md` covers:

- `claude -p` invocation and working-directory handling;
- model aliases (`fable`, `opus`, `sonnet`, `haiku`) and effort selection;
- prompt files, system prompts, `CLAUDE.md`, skills, and bare mode;
- text, JSON, stream-JSON, and JSON Schema output;
- permission modes, restricted execution, and tool allowlists;
- diff-driven review without changing the reviewed tree;
- session IDs, resume, and fork; and
- process ownership for long-running jobs and interrupted runs.

## Install

Place this directory in the skill directory used by Claude Code, keeping the
directory name `claude-run`. Claude Code must already be installed and
authenticated; this skill does not install or authenticate the CLI.

The CLI is versioned. When a command or option is uncertain, check:

```bash
claude --help
```

The license is Apache-2.0; see [LICENSE](LICENSE).
