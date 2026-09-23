# DSH Tool Mapping

You are running in the **DeepSeek Harness (DSH)**. Superpowers skills describe
*actions*, never concrete tools; this file translates those actions into the
tools DSH actually gives you. Trust your own live tool list over this table
when they disagree — the harness can add or scope tools per session.

## Core mapping

| Action a skill names | DSH tool |
|---|---|
| Read a file | `read` (line-numbered; use `offset`/`limit` for long files) |
| Create or fully replace a file | `write` |
| Edit a file (targeted literal replacement) | `edit` (`old_string` must match exactly once; `replace_all` for repeats) |
| Run a shell command | `pwsh` (PowerShell). Pass `workdir` instead of `cd`; each call is a fresh process. Use `run_in_background: true` for long-running commands |
| Find files by name / path pattern | `glob` |
| Search file contents | `grep` (ripgrep regex; `include` filters by filename glob) |
| Fetch a URL | `web_fetch` |
| Web search | `web_search` |
| **Create / update todos** | `todo_write` — send the **entire** list every call; it replaces the previous list |
| **Invoke a skill** | `skill` with the **exact bare name** from the session skill catalog, e.g. `skill` → `brainstorming` |
| **Dispatch a subagent** | `subagent` (see below) |
| Read a background job's output | `job_output` |
| Ask your human partner a structured question | `ask_user_question` |

## Skill names: drop the `superpowers:` prefix

Skills refer to each other as `superpowers:brainstorming`. **DSH skill names are
bare** — the `skill` tool takes `brainstorming`, not `superpowers:brainstorming`.
The command gesture is `/brainstorming`. Passing a prefixed name fails with
`unknown or no longer available`.

## Todos

`todo_write` replaces the whole list on every call, so always send every item
with its current status. Use `in_progress` for what you are actively doing and
mark items `completed` the moment they finish. Skip the list entirely for
trivial single-step work.

## Dispatching subagents

DSH ships first-class subagent dispatch, so `dispatching-parallel-agents` and
`subagent-driven-development` run at full strength here.

| Need | Tool |
|---|---|
| Dispatch a fresh, self-contained worker | `subagent` — **runs in the background by default** and returns a durable agent id immediately |
| Dispatch a worker that inherits this conversation | `subagent_fork` |
| Block until that worker's result is needed | `subagent` with `run_in_background: false` |
| Message / steer a running or idle worker | `send_message` |
| Cancel a worker's current turn | `interrupt_agent` |
| List your workers and their status | `list_agents` |
| Discover routable provider/model ids | `list_subagent_models` |

Notes that matter for the dispatch skills:

- **There is no `Task` tool.** If you catch yourself about to call `Task`, call
  `subagent` instead. Never fabricate `Task` calls.
- **Completion is pushed, not polled.** When a background subagent settles, the
  runtime notifies you with its outcome and final message. Do not busy-poll or
  sleep waiting for it — start independent work instead, and collect results
  with `job_output` only when you are genuinely blocked (or set
  `run_in_background: false` up front).
- **Model routing is optional and inherited by default.** Omit `provider`,
  `model`, and `reasoning_effort` to inherit a compatible route from your own
  agent. Supply `provider` and `model` **together** to route deliberately;
  changing the route without naming an effort uses that model's default effort.
  A machine-level default may already be configured — check
  `list_subagent_models` and the session's configured child defaults before
  assuming you must choose models on every dispatch.
- Pass each child a **complete, standalone prompt**: it does not see this
  conversation. Use `subagent_fork` when the child genuinely needs this
  conversation's context.

## Shell notes (Windows)

`pwsh` is the shell tool. Skills that shell out to bash — notably
`subagent-driven-development`'s helper scripts — should be invoked through Git
Bash when available:

```powershell
bash -lc './scripts/sdd-workspace docs/superpowers/plans/my-plan.md'
```

If Git Bash is unavailable, read the helper script and perform its documented
effect by hand rather than reporting the skill as unrunnable.

Long-running commands (a dev server, a watch process, a visual-companion
server) must use `run_in_background: true` on the `pwsh` call so they survive
across turns; then read the returned job id with `job_output`.

## Platform-specific skills

- `brainstorming`'s optional visual companion starts a local HTTP server. On
  Windows, launch it with `pwsh` + `run_in_background: true`, then read the
  server-info file it writes to learn the URL and port.

## What is deliberately *not* mapped

- `workflow` — a scripted multi-agent orchestration tool. Use it only when your
  human partner explicitly asks for a workflow or for large fan-out; ordinary
  delegation is `subagent`.
- `ralph` — fresh-agent iterative loops. Only when your human partner
  explicitly asks for a Ralph loop.
- Goal tools (`create_goal`, `get_goal`, `update_goal`) — for one long-running
  same-session objective, not for skill-driven task tracking. Use `todo_write`
  and the plan/ledger files the skills describe.
