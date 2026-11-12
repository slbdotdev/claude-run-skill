---
name: claude-run
description: Run Claude Code non-interactively from another agent or automation harness, including model and effort selection (Fable, Opus, Sonnet, Haiku), CLAUDE.md and skills, permission modes, structured output, streaming, session resume, reviews, and long-running jobs.
---

# Run Claude Code headlessly

Use `claude -p` (`--print`) for one non-interactive run. Claude works in the
caller's current directory, so change directory explicitly:

```bash
(cd /path/to/repo && \
  claude -p \
    --model fable \
    --effort high \
    --output-format json \
    < /tmp/claude-brief.md \
    > /tmp/claude-result.json)
```

The launching machine needs an installed, authenticated `claude`; this skill
does neither. In a service runner use the absolute path from
`command -v claude`, because service managers often lack the shell's `PATH`.
Flags change between releases: confirm with `claude --help` and
`claude --version` before relying on one.

## Model and effort

`--model` takes an alias for the latest model in a family or a full model ID:

| Alias | Use for |
| --- | --- |
| `fable` | Hardest reasoning and long-horizon agentic work; the most capable and most expensive tier. Currently Claude Fable 5.2 (`claude-fable-5-2`). |
| `opus` | Strong general default for features, fixes, and review. |
| `sonnet` | Fast everyday coding and worker or subagent runs. |
| `haiku` | Cheap, high-volume, mechanical tasks. |

Prefer aliases. Pin a full ID only when reproducibility matters, and take it
from the provider's current model list; IDs differ on Bedrock, Vertex, and
Foundry.

`--effort low|medium|high|xhigh|max` sets reasoning depth and token spend:

- `low` for quick questions and mechanical edits; `medium` for routine work;
- `high` for complex or high-risk work, and as the starting point on Fable;
- `xhigh` or `max` only where a measured gain justifies the cost.

Fable specifics:

- Turns on hard tasks can run many minutes. Set harness timeouts accordingly
  and use `stream-json` when progress must be visible.
- Give the full task, constraints, and acceptance criteria up front, and state
  goals rather than step-by-step procedure; over-prescriptive briefs written
  for older models reduce output quality.
- Safety classifiers can decline a request. Treat a refusal as a result to
  report, not a transient error to retry blindly.
- Set `--max-budget-usd` on API-billed runs.

`--fallback-model <model[,model]>` switches models when the primary is
overloaded or unavailable. Use it only deliberately and record it; never
silently fall back to another model or provider.

## Prompts and context

Put the brief in a file and pipe it on stdin (`claude -p < brief.md`). State:

- the task, scope, and working directory;
- files or directories that must not change;
- required checks or acceptance criteria; and
- the final report expected, including anything left unverified.

Stdin is capped at 10 MB; for larger context, put the data in the workspace
and name the file to read.

- `--append-system-prompt` adds instructions to the default system prompt;
  `--system-prompt` replaces it entirely.
- `--add-dir` grants access to additional directories.
- Without `--bare`, print mode loads the same `CLAUDE.md`, settings, hooks,
  MCP servers, and skills as an interactive session. Invoke a skill with
  `/skill-name` in the prompt; `--disable-slash-commands` turns skills and
  commands off.
- `--settings <file-or-json>` overlays settings for one run.
- `--bare` skips hooks, plugin sync, auto-memory, keychain reads, and
  `CLAUDE.md` discovery for reproducible CI-style runs. It never reads OAuth
  credentials: supply `ANTHROPIC_API_KEY`, an `apiKeyHelper`, or third-party
  provider credentials, and pass any needed context explicitly
  (`--append-system-prompt`, `--add-dir`, `--mcp-config`, `--settings`).
- `--safe-mode` disables all customizations to troubleshoot a broken
  configuration. It is a diagnostic, not a worker default.

## Output

- `--output-format text` (default) prints the final response.
- `--output-format json` prints one result object with the final text,
  `session_id`, and usage and cost metadata. Check the exit status too: a
  failed run can still print a result.
- `--output-format stream-json` prints newline-delimited events ending in a
  `result` event. Add `--verbose --include-partial-messages` for token
  deltas, and `--forward-subagent-text` for subagent transcripts
  (distinguished by `parent_tool_use_id`).
- `--json-schema '<schema>'` validates structured output; use it with JSON
  output and read the `structured_output` field.

```bash
claude -p "Extract the risky changes" \
  --output-format json \
  --json-schema '{"type":"object","properties":{"risks":{"type":"array","items":{"type":"string"}}},"required":["risks"]}' \
  > /tmp/claude-risk.json
```

## Permissions

Nobody answers prompts in an unattended run, so set policy explicitly and
grant the least authority that completes the task:

- `--permission-prompts none` denies anything that would prompt instead of
  waiting on a permission host.
- `--permission-mode plan` for inspection and planning; `acceptEdits`
  auto-approves file edits only; `auto` uses Claude's classifier; `dontAsk`
  denies whatever is not pre-approved.
- `--tools "Read,Glob,Grep"` limits the built-in tool set.
  `--allowedTools "Read,Edit,Bash(git diff *)"` pre-approves specific calls;
  `--disallowedTools` denies them. MCP tools need their own rules.
- `--restricted` removes command-running tools and WebFetch, ignores user,
  project, and local settings, confines file tools to the working
  directories, and refuses bypass.
- `--dangerously-skip-permissions` bypasses every check. Use it only in a
  disposable, externally isolated sandbox.

For a controlled editing run, prefer `acceptEdits` plus an allowlist over a
global bypass.

## Review a diff

Pipe the exact diff so the reviewer needs no Bash access:

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

Use an explicit range such as `git diff origin/main...HEAD` when that is the
scope. Prefer a reviewer from a different model family than the author; if
that is impossible, use a fresh session and label the review as weaker.
`claude ultrareview` is a separate cloud-hosted multi-agent review; see
`claude ultrareview --help`.

## Sessions

Sessions persist by default, and the session ID is the durable handoff key
(not a process ID):

```bash
session_id=$(claude -p "Start the investigation" \
  --output-format json | jq -r '.session_id')
claude -p "Continue the investigation and verify the result" \
  --resume "$session_id" \
  --output-format json
```

`--resume` takes a session ID or name; `--continue` resumes the most recent
conversation in the current directory; `--fork-session` gives the follow-up a
new ID. Avoid `--no-session-persistence` for work that may need audit or
continuation.

## Long-running jobs

`claude -p` blocks until the run finishes. If the calling harness reaps its
children, launch the run under an owner it does not control (systemd,
`launchd`, a terminal multiplexer, a container):

```bash
systemd-run --user --unit=claude-<job> --collect /path/to/runner.sh
```

The runner should call `claude` by absolute path and write prompts and
results to inspectable files. Do not start long-lived processes from inside
the run: Claude terminates background Bash tasks shortly after a print run
returns. Subagents are different; print mode waits for their results.

`claude --bg` instead starts a detached interactive-style session and prints
an ID for `claude attach`, `logs`, `stop`, and `rm`.

## Failed or interrupted runs

Claude may have edited files before a run stopped. Inspect before retrying:

```bash
git status --short
git diff
```

Run the relevant checks yourself, then resume the recorded session or write a
new brief that accounts for the edits. A run killed with SIGTERM exits 143
with its turn unfinished; resume it rather than assuming no progress. For
authentication, quota, or model failures, report the exact invocation and
error.
