---
name: claude-run
description: Run Claude Code non-interactively from another agent or automation harness, including prompt files, permissions, models and effort, structured output, streaming, session resume, reviews, and long-running jobs.
---

# Run Claude Code headlessly

Use `claude -p` (or `claude --print`) for one non-interactive run. Claude uses
the caller's current working directory; there is no Codex-style `-C` flag, so
change directory explicitly in the runner:

```bash
(cd /path/to/repo && \
  claude -p \
    --model <model> \
    --effort medium \
    --output-format json \
    < /tmp/claude-brief.md \
    > /tmp/claude-result.json)
```

The machine that launches the worker needs an installed and authenticated
`claude` command. This skill does not install or authenticate Claude Code.
Find the executable with `command -v claude` and use its absolute path in a
service runner; service managers often do not inherit the interactive shell's
`PATH`.

Check the installed CLI before relying on a flag:

```bash
claude --help
claude --version
```

## Prompts and context

Prefer a brief in a file. It should state:

- the task, scope, and working directory;
- files or directories the worker must not change;
- required checks or acceptance criteria; and
- the required final report, including anything it could not verify.

Print mode reads stdin, so `claude -p < brief.md` is the normal file-input
form. Stdin is capped at 10 MB; for larger context, put the data in the
workspace and tell Claude which file to read rather than piping it wholesale.
Use one prompt input method and keep the output on a separate file descriptor
or redirection.

Use `--append-system-prompt` to add a worker instruction while retaining
Claude Code's normal system prompt. Use `--system-prompt` only when replacing
the entire default prompt is intentional. `--add-dir` grants access to
additional working directories.

For deterministic CI-style execution, `--bare` skips host hooks, plugins,
MCP, auto memory, and `CLAUDE.md` discovery. It also does not use OAuth or the
keychain, so provide an API key or explicit provider credentials. Do not use
`--bare` when the run is supposed to consume the installed skill or host
configuration unless you provide that context explicitly.

## Choose output and input modes

- `--output-format text` is the default plain final response.
- `--output-format json` emits one JSON result with the final text, session ID,
  usage/cost metadata, and other run metadata. Check the exit status as well:
  a run-time failure can still print a failure result on stdout.
- `--output-format stream-json` emits newline-delimited events. Add
  `--verbose` and `--include-partial-messages` when a consumer needs token
  deltas; the final `result` event is the completion record.
- `--json-schema '<schema>'` validates structured output and requires
  `--output-format json`. Read the model's structured value from the
  `structured_output` field, not from streaming text deltas.
- `--input-format stream-json` is for a caller that sends a realtime message
  stream; ordinary briefs should use text stdin.

Example structured result:

```bash
claude -p "Extract the risky changes" \
  --output-format json \
  --json-schema '{"type":"object","properties":{"risks":{"type":"array","items":{"type":"string"}}},"required":["risks"]}' \
  > /tmp/claude-risk.json
```

## Permissions and isolation

Print mode is non-interactive, but a tool that would prompt still needs an
explicit policy. Choose the least authority that can complete the task:

- `--restricted` removes code-running tools and ignores user, project, and
  local settings; it confines file tools to the working directories and
  refuses bypass permissions.
- `--tools "Read,Glob,Grep"` restricts the built-in tool set for read-only
  probes. `--disallowedTools` adds explicit denials; MCP tools need their own
  restrictions.
- `--permission-mode plan` is a good starting point for inspection and
  planning. `acceptEdits` auto-approves file edits while leaving other actions
  subject to permissions. `auto` uses Claude's classifier. `dontAsk` denies
  anything that would require a person.
- `--permission-prompts none` prevents an unattended run from waiting for a
  permission host; requests not otherwise allowed are denied. It is available
  in current Claude Code releases, but check help on older installations.
- `--dangerously-skip-permissions` bypasses all checks. Use it only in a
  disposable, externally isolated sandbox; it is not a normal automation
  default.

For a read-only review, combine a restricted tool set with a prompt that
forbids edits. For a controlled editing run, prefer `acceptEdits` plus an
allowlist such as `--allowedTools "Read,Edit,Bash(git diff *)"` over a global
bypass.

## Review changes

Claude Code's local CLI has no Codex-style `exec review --base/--commit`
selector. Review the exact diff by supplying it as input and ask for findings:

```bash
git diff --cached --no-ext-diff | \
  claude -p \
    --append-system-prompt "You are an independent code reviewer. Do not edit files. Return prioritized findings with file and line references." \
    --permission-mode plan \
    --permission-prompts none \
    --output-format json \
    "Review the supplied diff for correctness, security, and missing tests." \
    > /tmp/claude-review.json
```

Use `git diff origin/main...HEAD` or another explicit Git range when that is
the review scope. Piping the diff means Claude does not need Bash permission
to discover it. Keep the review model independent from the author's family
where possible; if that is not possible, use a fresh blind run and label it
the weaker form of review.

The installed CLI also exposes `claude ultrareview`; it is a separate
cloud-hosted multi-agent workflow. Check `claude ultrareview --help` before
using it, and do not confuse it with the local, deterministic diff pattern
above.

## Sessions and resume

Sessions persist by default. Do not use `--no-session-persistence` for work
that may need audit or continuation: it disables the on-disk record and makes
the session non-resumable.

Capture the session ID from JSON output and resume deliberately:

```bash
session_id=$(claude -p "Start the investigation" \
  --output-format json | jq -r '.session_id')
claude -p "Continue the investigation and verify the result" \
  --resume "$session_id" \
  --output-format json
```

`--continue` resumes the most recent conversation in the current project;
`--resume` accepts a session ID, name, or transcript path. Use
`--fork-session` when the follow-up should receive a new session ID. The
session ID is the durable handoff key; do not identify a run by a process ID.

## Long-running jobs

`claude -p` blocks until the print run finishes. The interactive `--bg`/
`--background` option starts a background session and returns an ID, but it is
not compatible with `-p`; manage such sessions with `claude attach`, `logs`,
`stop`, and `rm` only when an interactive background session is what you want.

If the calling harness reaps its own child processes, launch the print run
under an owner the harness does not control. On Linux with systemd:

```bash
systemd-run --user --unit=claude-<job> --collect /path/to/runner.sh
```

The runner should call Claude by its absolute path and write prompts and
results to inspectable files. Claude also terminates background Bash tasks a
few seconds after a print run returns; a child started inside Claude is not a
reliable job supervisor. Put the long-lived process outside Claude under
systemd, `launchd`, a terminal multiplexer, a container, or another durable
owner. Subagents and workflows are different: print mode waits for their
results, subject to Claude's configured wait ceiling.

## Failed or interrupted runs

If a run stops before producing a trustworthy report, inspect the workspace
before retrying:

```bash
git status --short
git diff
```

Claude may have edited files before its final result was emitted. Run the
relevant checks yourself and resume the existing session or start a new brief
that accounts for the edits. A process killed with SIGTERM exits 143 and can
leave its current turn unfinished; resume the recorded session instead of
assuming it made no progress.

For authentication, quota, or model failures, report the exact invocation and
do not silently fall back to a different provider or model family. Use
`--fallback-model` only when that change is deliberate and recorded.

## Choose a model and effort

Aliases such as `opus`, `sonnet`, and `haiku` move as Claude Code updates.
Use an exact model string when reproducibility matters and confirm that the
account or provider exposes it:

| Common name | Exact model string | Recommended starting effort | Best fit |
| --- | --- | --- | --- |
| Opus | `claude-opus-5` | `high` | hardest, highest-risk work |
| Sonnet | `claude-sonnet-5` | `medium` | everyday features, fixes, tests, review |
| Haiku | `claude-haiku-4-5-20251001` | `low` | fast questions and mechanical edits |

These are starting points, not guarantees. Effort is model-dependent and can
be set with `--effort low|medium|high|xhigh|max`; lower effort is usually the
cost/latency lever, while higher effort is for work where additional reasoning
is worth the time. A model alias may resolve differently after an update, so
refresh `claude --help` and the provider's current model list before pinning a
new deployment.

For independent review, prefer a model family different from the author where
one is available. Record any same-family fallback honestly.
