---
name: claude-run
description: Run Claude Code non-interactively (`claude -p`) from another agent, script, or CI job. Covers prompts and stdin, CLAUDE.md and skills, bare mode, permissions, JSON, streaming, and structured output, session IDs and resume, diff review, long-running jobs, and model and effort choice.
---

# Run Claude Code headlessly

`claude -p` (long form `--print`) runs one task without a person and exits.
Claude works in the current directory of the process that starts it, so set
the directory in the runner:

```bash
(cd /path/to/repo && \
  claude -p --model sonnet --effort medium --output-format json \
    < brief.md > result.json)
```

Before you rely on it:

- The machine needs an installed, signed-in `claude` command. This skill does
  not install or authenticate it.
- In a service, call Claude by the absolute path from `command -v claude`.
  Service managers often do not inherit the shell's `PATH`.
- Flags change between releases. Check `claude --version` and `claude --help`.
  Version numbers below mark when a flag or behavior first appeared.
- `-p` skips the workspace trust dialog. Without `--bare`, it runs the
  project's hooks and connects the MCP servers in its `.mcp.json`, even in a
  folder nobody has trusted. Run it only in directories you trust.
- In `-p` mode, a settings file that fails validation is ignored without an
  error.

## Prompts and input

Write the brief to a file. It should state:

- the task, the scope, and the working directory;
- files or directories the worker must not change;
- the checks or acceptance criteria; and
- the final report you want, including anything it could not verify.

There are three ways to pass the prompt:

- `claude -p < brief.md` reads the prompt from stdin.
- `claude -p "prompt"` takes the prompt as an argument.
- `cat data.txt | claude -p "instruction"` uses both. Claude receives the
  argument first, then the stdin text.

Piped stdin is capped at 10 MB. Above that, `claude` exits with an error. For
larger input, write the data to a file in the workspace and name the path in
the prompt.

Some flags take a list of values: `--tools`, `--allowedTools`,
`--disallowedTools`, `--add-dir`, and `--mcp-config`. A prompt argument placed
after one of them is read as another list item, and the run fails with
"Input must be provided". Put the prompt before those flags, or after `--`:

```bash
claude -p --tools "Read,Grep,Glob" -- "Summarize the README"
```

To change the system prompt:

- `--append-system-prompt "<text>"` or `--append-system-prompt-file <file>`
  adds instructions and keeps Claude Code's default prompt. Use this normally.
- `--system-prompt` or `--system-prompt-file` replaces the whole default
  prompt. Use it only when that is intended.

`--add-dir <dir>` gives the run access to directories outside the working
directory.

## Project context, skills, and bare mode

Without `--bare`, `claude -p` loads what an interactive session loads from the
working directory and `~/.claude`: `CLAUDE.md`, settings, hooks, plugins, MCP
servers, subagents, skills, and auto memory.

- Put `/skill-name` in the prompt to run a user-invoked skill.
- `--disable-slash-commands` turns off all skills and commands for the run.
- `--settings <file-or-json>` adds settings for one run.
  `--setting-sources user,project,local` picks which settings files load.
- `--mcp-config <file>` with `--strict-mcp-config` uses only the MCP servers
  you name.

`--bare` skips that discovery. Use it in CI when every machine should start
the same way. Claude Code's docs recommend it for scripted calls and say it
will become the `-p` default in a future release. In bare mode:

- Anthropic API auth comes only from `ANTHROPIC_API_KEY` or an
  `apiKeyHelper` in `--settings`. OAuth sign-in and the keychain are never
  read. Bedrock, Google Cloud, and Microsoft Foundry still use their own
  credentials.
- Pass the context you need with flags: `--append-system-prompt[-file]`,
  `--settings`, `--mcp-config`, `--agents`, `--plugin-dir`, and `--add-dir`.
  Bare mode loads skills from an added directory's `.claude/skills/`.
- `/skill-name` in the prompt still runs a skill.
- Claude cannot run background commands or background subagents.

`--safe-mode` is for troubleshooting a broken setup. It turns off `CLAUDE.md`,
skills, plugins, hooks, MCP servers, and other customizations, but keeps
normal sign-in. It is not a worker default and is not the same as `--bare`.

## Output

- `--output-format text` (the default) prints only the final response.
- `--output-format json` prints one JSON object. `result` holds the final
  text. Other fields include `session_id`, `is_error`, `subtype`, usage, and
  `total_cost_usd`, which is a client-side estimate. Check both the exit
  status and `is_error`.
- `--output-format stream-json` prints one JSON event per line. With `-p` it
  requires `--verbose`. Add `--include-partial-messages` for token-by-token
  deltas. The final `result` event is the completion record.
- In the stream, messages from subagents carry the `parent_tool_use_id` of
  the call that started them. Main-conversation messages carry `null`.
  `--forward-subagent-text` also forwards subagents' text and thinking. It
  works only with stream-json.
- `--input-format stream-json` is for a caller that sends messages as a live
  stream. An ordinary brief uses plain text.

For structured output, pass a JSON Schema with `--json-schema` and use
`--output-format json`. Read the value from `structured_output`:

```bash
claude -p "List the risky changes in this repo" \
  --output-format json \
  --json-schema '{"type":"object","properties":{"risks":{"type":"array","items":{"type":"string"}}},"required":["risks"]}' \
  | jq '.structured_output'
```

If the run ends without valid structured output, for example because Claude
declined, `structured_output` is missing. Check that it exists before you use
it. An invalid schema makes `claude` exit with an error.

## Permissions

No person answers permission prompts in a `-p` run. A tool call that needs
approval is denied unless a permission rule, a hook, the permission mode, or a
permission host allows it. A permission host is an Agent SDK `canUseTool`
callback or an MCP tool named with `--permission-prompt-tool`. From v2.1.302,
a `-p` run starts in Manual mode when nothing sets a mode. Give the run the
least authority that can finish the task:

- `--tools "Read,Grep,Glob"` makes only those built-in tools exist. Use it for
  read-only work. It does not affect MCP tools; add
  `--disallowedTools "mcp__*"` to block those too.
- `--allowedTools "Edit,Bash(npm test *)"` pre-approves tools or commands.
  The space before `*` matters: `Bash(git diff *)` matches commands that start
  with `git diff `, but `Bash(git diff*)` also matches `git diff-index`.
- `--disallowedTools` denies tools.
- `--permission-mode` sets the baseline:
  - `manual` (config value `default`): edits, most shell commands, and
    network access need approval, so in `-p` they are denied unless allowed
    another way.
  - `plan`: Claude reads and plans, but does not edit.
  - `acceptEdits`: file edits and common file commands (`mkdir`, `touch`,
    `mv`, `cp`) are approved. Other shell commands and network access still
    need allow rules.
  - `dontAsk`: anything that would prompt is denied. Allow rules and
    read-only actions still run. Suits locked-down CI.
  - `auto`: a classifier model reviews each action.
- `--permission-prompts none` (v2.1.259+) matters when the run has a
  permission host. Without it, the run waits for that host. With
  it, those requests are denied and Claude is told not to retry. In a plain
  `-p` run with no host, such requests are denied anyway, and the flag only
  stops retries.
- `--restricted` (v2.1.248+) is for a harness on a shared machine. It removes
  tools that run commands or code, and WebFetch, unless you name them in
  `--tools`. It ignores user, project, and local settings files. Managed
  settings and `--settings` still apply. It confines file tools to the
  working directories and refuses `bypassPermissions`. MCP servers still load
  unless you add `--strict-mcp-config`.
- `--max-turns <n>` and `--max-budget-usd <amount>` limit a run's length and
  spend. The budget uses a client-side estimate, includes subagents, and can
  overshoot.
- `--dangerously-skip-permissions` turns off every check. Use it only in a
  disposable sandbox with no network access.

For an editing run, prefer `--permission-mode acceptEdits` plus a narrow
`--allowedTools` list over a global bypass.

## Review a diff

Pipe the diff in so Claude does not need shell access to find it. Give the
reviewer read-only tools:

```bash
git diff --cached --no-ext-diff | \
  claude -p "Review this diff for correctness, security, and missing tests." \
    --append-system-prompt "You are an independent code reviewer. Do not edit files. Return prioritized findings with file and line references." \
    --tools "Read,Grep,Glob" \
    --output-format json \
    > review.json
```

For a branch, use an explicit range such as `git diff origin/main...HEAD`.

A fresh session shares no conversation with the author's session. It is still
the same model family. If the author was also Claude, say so in the report.

`claude ultrareview [target]` is a different tool. It sends the current
branch, a PR number, or a base branch to a cloud-hosted multi-agent review.
Check `claude ultrareview --help` before using it.

## Sessions and resume

Sessions are saved to disk by default. `--no-session-persistence` turns that
off, and the run then cannot be resumed or audited. Do not use it for work
that might need follow-up.

Choose the session ID yourself. The caller then knows it before the run
starts, even if the run crashes:

```bash
session_id=$(uuidgen)
claude -p --session-id "$session_id" --output-format json \
  < brief.md > result.json
claude -p "Continue and verify the result" \
  --resume "$session_id" --output-format json > followup.json
```

- When the caller is itself a Claude Code session, the child can inherit the
  parent's session ID from the environment. `--session-id` prevents that.
- `--resume` accepts a session ID, a session name, or the absolute path to
  the session's `.jsonl` transcript. It finds an ID in any project on the
  machine.
- `--continue` resumes the most recent conversation in the current directory.
- `--fork-session` with `--resume` or `--continue` gives the follow-up a new
  session ID and leaves the original unchanged.
- On resume, new system prompt flags have no effect at first. Claude Code
  reuses the prompt recorded on the first request until the conversation is
  compacted. Pass `--system-prompt-snapshot off` to rebuild it on every
  request.
- After a resume, `total_cost_usd` covers the whole conversation, including
  earlier runs.

Identify a run by its session ID, not by its process ID.

## Long-running jobs

`claude -p` blocks until the task finishes. After Claude's last turn, it also
waits for background work that Claude started:

- background commands, until they exit or hit their time limit; Claude then
  takes one more turn with the outcome;
- background subagents and workflows, until they finish; and
- other idle waiting, for at most 10 minutes. Set
  `CLAUDE_CODE_PRINT_BG_WAIT_CEILING_MS` to change the cap, or `0` for none.

It does not wait for MCP work running in the background. A `-p` run is not a
supervisor: start a service that must outlive the run outside Claude.

`--bg` starts an interactive background session that you manage with
`claude agents`, `attach`, `logs`, `stop`, and `rm`. It refuses to run with
`-p`. Use it only when you want a session you can attach to.

If the calling harness kills its child processes when it exits, start the
run under an owner the harness does not control. On Linux with systemd:

```bash
systemd-run --user --unit=claude-<job> --collect /path/to/runner.sh
```

`launchd`, `tmux`, or a container also work. The runner should call `claude`
by its absolute path and write prompts and results to files.

## Stopped or failed runs

- SIGTERM makes `claude` exit with code 143. The current turn stays unfinished
  with no result, and any running Bash command's process tree is killed. Send
  SIGINT first if you want the turn to end cleanly.
- On resume, Claude leaves an interrupted turn as it is, and your next prompt
  drives the conversation. Set `CLAUDE_CODE_RESUME_INTERRUPTED_TURN=1` to have
  it continue that turn instead.
- Claude may have edited files before it stopped. Before you retry, run
  `git status --short` and `git diff`, and run the relevant checks yourself.
- For sign-in, quota, or model errors, report the exact command. Do not
  silently switch to another provider or model. Use
  `--fallback-model <model>[,<model>...]` only on purpose, and record it.
  The fallback list is used when the primary model is overloaded or
  unavailable, and the primary is tried again at each new user turn.

## Model and effort

- `--model` takes an alias or a full model ID. Aliases include `fable`,
  `opus`, `sonnet`, and `haiku`. Each points to the latest model of that
  kind, so its target changes over time. To pin a model for repeatable runs,
  take the full ID from your provider's current model list, not from an old
  example.
- `--effort low|medium|high|xhigh|max` sets how much Claude reasons. The
  levels available depend on the model. Start with `low` for quick questions
  and mechanical edits, `medium` for ordinary features, fixes, tests, and
  docs, and a higher level for complex, high-risk, or long work.
- Pricing, usage limits, and which aliases exist vary by provider and plan.
  Check the provider's current docs before choosing a setting for every
  deployment.
