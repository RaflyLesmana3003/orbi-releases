# Agent widgets: the contract

Orbi shows AI agent requests on a small island. The black hole is Orbi, each
agent is a planet, and its reported subagents are moons. This document describes
the MCP bridge (`windows/mcp/`), Rust app (`windows/src-tauri/src/agents/`) and
TypeScript island (`windows/src/`).

## 1. Topology

```text
agent session
  │ MCP over stdio (JSON-RPC)
  ▼
orbi-mcp                  one process per session
  │ UTF-8 JSON lines over a local transport
  ▼
Orbi (Tauri, Rust) ── events ──▶ island webview
```

- Windows: `\\.\pipe\orbi-agents-<sid>`. The bridge checks that the server process
  belongs to the same user.
- Linux: `$XDG_RUNTIME_DIR/orbi-agents.sock`, falling back to
  `/run/user/<uid>/orbi-agents.sock`.
- macOS: `~/Library/Application Support/Orbi/orbi-agents.sock`.
- On both, the socket's folder must be private and owned by the user, the app
  makes the socket 0600, and each side checks the other's peer credentials.
- MCP initialization records the client name/version. The first tool call opens
  the app connection; merely starting a configured server creates no planet.
- One connection drives one session. Disconnect removes session widgets,
  questions and subagents. Persistent or expiring widgets retain an ended session.
- Finished or failed sessions silent for 20 minutes, working ones silent for two hours, end on the minute sweep; a session with a pending question stays. A subsequent request
  receiving `no-session` replays the session and retries once.
- Missing app: `{"ok":false,"error":"orbi-not-running"}`. Each call while
  disconnected makes one connection attempt, waiting at most three seconds;
  there are no background retries, so an agent that stays idle while Orbi
  restarts does not reappear. Its next tool call reconnects and replays
  `hello`, the last `introduce`, the last `status`, last widget per id and
  running subagents. Notifications and questions are not replayed.
- Writes and ordinary replies each have a three-second timeout. A stalled write
  triggers reconnection; timeout results use `{"ok":false,"error":"timeout"}`.
  JSON lines have a 256 KiB transport limit.

## 2. Wire protocol: bridge ⇄ app

One JSON object per line. The bridge adds a string `request_id` to tool requests;
the app also accepts numeric ids. Replies echo it. `hello` and reconnect replay
lines may omit it.

| Bridge → app `type` | Fields |
|---|---|
| `hello` | `session_id`, `client?: {name,version}`, `pid?`; first line |
| `introduce` | `name`, `task?` |
| `widget.set` | `widget: {id,type,title,data,lifetime}` (§4) |
| `widget.remove` | `id` |
| `widget.list` | No additional fields |
| `status` | `state`, `message?` |
| `notify` | `message`, `level?` |
| `clarify` | `question`, `detail?`, `options: [{label,description?}]`, `multi`, `allow_text`, `placeholder?`, `timeout_ms` |
| `clarify.cancel` | `request_id` of the question to remove |
| `subagent` | `id`, `state`, `label?`, `parent?` |

| App → bridge `type` | Fields |
|---|---|
| `ack` | `request_id?`, `ok`, `error?` |
| `widget.list.result` | `request_id?`, `widgets: [{id,type,title,lifetime}]`; replaces the ordinary ack |
| `clarify.answer` | `request_id`, `answered`, `choices?: [label]`, `text?`, `reason?` |

A question receives an immediate ack, then a later `clarify.answer`.
One question may wait per session. Replacement answers the old one with
`reason: "replaced"`; skip returns `"dismissed"`, expiry returns `"timeout"`.
Choices are checked against the original options; typed answers are clipped to
1000 characters. The bridge waits the requested timeout plus five seconds,
then cancels the card and returns an unanswered timeout.

The app accepts at most 64 sessions, 256 widgets total, 32 widgets per session,
and eight subagents per session. New entries over these limits return
`too-many-sessions`, `too-many-widgets` or `too-many-subagents`.
A notification within two seconds of the previous one returns `rate-limited`.
A new subagent may evict the oldest finished entry; eight running entries refuse
a ninth. Session ids are nonempty, at most 64 bytes, without control characters.
A second `hello` with a different id ends the connection's previous session.

## 3. MCP tools

Server info name and agent config key: `orbi`. Version: the bridge crate version.
Claude Code and Codex expose tools as `mcp__orbi__*`; Gemini uses `mcp_orbi_*`.

All eight tools are declared in `mcp/src/server.rs` and validated by
`mcp/src/tools.rs`. Optional fields may be omitted or null. Text is trimmed,
stripped of control characters and clipped by Unicode character count. Invalid
required fields, types, enums or ids return an MCP tool error; unknown keys are
discarded. Application refusals return `{ok:false,error}` as a normal tool result.

| Tool | Parameters | Result |
|---|---|---|
| `introduce` | Required `name` ≤40; optional `task` ≤140, `state`, `message` ≤80, `widgets`, `remove` | `{ok,error?}` |
| `show_widget` | Required `id`, `type`, `title` ≤80, `data`; optional `lifetime` (default `session`), `widgets`, `remove` | `{ok,error?}` |
| `remove_widget` | Required `id`; optional `remove` | `{ok,error?}` |
| `list_widgets` | None | `{ok,widgets:[{id,type,title,lifetime}]}` |
| `set_status` | Required `state`; optional `message` ≤80, `widgets`, `remove` | `{ok,error?}` |
| `notify` | Required `message` ≤120; optional `level` (default `info`) | `{ok,error?}` |
| `clarify` | Required `question` ≤200; optional `detail` ≤300, `options` ≤6, `multi`, `allow_text`, `placeholder` ≤60, `timeout_s` | `{ok,answered:true,choices,text?}` or `{ok,answered:false,reason}`; transport errors use `{ok:false,error}` |
| `subagent` | Required `id`, `state`; optional `label` ≤60, `parent` | `{ok,error?}` |

`show_widget` declares every `data` field with its type, limit and the widget types that use it, and its description and the bridge's instructions say which widget fits what.

**Batches.** So that a turn takes few calls, `introduce`, `show_widget`, `remove_widget` and
`set_status` also take `remove`, up to eight widget ids to take down, and (all but
`remove_widget`) `widgets`, up to eight more widgets as `show_widget` takes them.
`introduce` also takes `state` and `message` as `set_status` does; a `message` needs a
`state`. The bridge checks the whole call before sending anything: one invalid part, or
a list over eight, is a tool error naming it (`widgets[2]: ...`) and nothing is sent.
It then sends the parts in order: removals, the tool's own message, the widgets, and the
status last. A part the app refuses stops the rest, and that refusal is the result.
Each part is an ordinary wire message, so the app needs nothing new.

A typical task takes `introduce` (task, `state: working`, the `progress` plan in
`widgets`), one `show_widget` per step change, and one `set_status` (`finished`, the
`brief` in `widgets`, the plan in `remove`).

- Widget and subagent ids (including `parent`): `^[a-z0-9_-]{1,64}$`.
  A subagent cannot be its own parent.
- Status states: `working`, `thinking`, `finished`, `error`.
  Subagent states: `running`, `finished`, `error`.
- Notification levels: `info`, `success`, `warning`.
- Question options are strings or `{label,description?}`; labels are distinct and
  nonempty, ≤60 characters; descriptions ≤120. Arrays are cut to six entries.
  `multi` defaults false and needs at least two options. `allow_text` defaults
  true only when no options remain; without options it must be true.
- `timeout_s` is a nonnegative integer, clamped to 10–600 seconds; default 60.
  The wire uses milliseconds. Codex installation sets `tool_timeout_sec = 620`.
- Call `introduce` first, with the project's folder/repository name and the
  user's task. On each new task, introduce again and restart progress. An
  `introduce` whose `task` differs from the last one first takes down every widget
  the session showed (the bridge's replay knows them; the same task again, or an
  unnamed one, removes nothing). Set `working` before work and `finished` or
  `error` at the end.
- A launched subagent shares its parent's MCP session. Its own `introduce`,
  status or widgets would overwrite the parent's. The parent reports it through
  `subagent`; nested children use `parent`. Subagents send their progress and
  questions to the parent.
- Ask questions with `clarify`. If unanswered, end the turn with the question
  instead of proceeding on a guess.

## 4. Widgets

`show_widget` replaces the widget with the same id **within its session**.
Different sessions may use the same id. The app adds `session_id`, `agent`
and millisecond `updated_at` fields.

| Type | Required data | Optional data and limits |
|---|---|---|
| `progress` | Numeric `value`, clamped to 0–1 | `label` ≤80; `steps` ≤8: `{name` ≤40, `state: done\|running\|todo}`; `stats` ≤4 with keys/values ≤32 |
| `stat` | String `value` ≤32 | `label` ≤80, `trend` ≤40, `trend_tone: up\|down\|neutral` |
| `list` | `items` ≤8: `{text` ≤120, `tone?: ok\|error\|neutral, detail?` ≤160`}` | `label` ≤80 |
| `brief` | `sections` ≤4: `{heading` ≤40, `items?: [string]}`, ≤6 items per section, each ≤120 | `link: {label?` ≤40, `url?` ≤2048`}`; label defaults to `Open` |

These are the MCP bridge's limits. Raw wire messages are revalidated by the app,
whose internal text limits are sometimes wider. Progress stats accept strings,
numbers or booleans and become strings. Oversized arrays and text are clipped;
unknown data keys are removed. Links beginning with `http://` or `https://` are
accepted; other schemes are dropped. All displayed text uses text nodes.

Lifetime is `"session"` (default), `"persistent"`, or
`{"expires":"<ISO-8601 date-time>"}`. Invalid or already elapsed expiration
dates are refused by the app. Persistent/expiring widgets survive disconnect and
restart; session widgets do not.

## 5. App → island events and commands

- `agents-changed`: full `{sessions,widgets}` snapshot after changes.
  Sessions carry `id,name,task?,client,avatar,state,message?,started_at,ended_at?`,
  optional `question`, and `moons`. Avatar defaults to a randomly chosen palette
  hue avoiding live duplicates when possible, with `eyes: "pill"`.
- Moons carry `id,label,state,started_at,ended_at?,parent?`, oldest first.
  Finished/error entries linger for 60 seconds before pruning; the minute sweep
  can leave them for nearly two minutes. A subagent update also prunes expired
  entries. A new `introduce` removes ended moons; running ones remain. Session
  end removes all moons. They are not saved to disk.
- `agent-notify`: `{session_id,message,level}`.
- `agents_snapshot()`: initial snapshot.
- `widget_dismiss(id,session_id?)`: remove a widget; without a session id, remove
  matching ids in all sessions.
- `session_forget(id)`: remove an ended session and its widgets.
- `clarify_answer(session_id,question_id,choices,text?,skipped)`: validate and
  return the user's response to the waiting bridge.

`%LOCALAPPDATA%\Orbi\agents.json` (Linux: `~/.local/share/orbi/agents.json`,
macOS: `~/Library/Application Support/Orbi/agents.json`)
stores durable widgets and their owning sessions, atomically when the saved
slice changes. Restart restores these sessions ended and `finished`, with no
questions or moons. Expired widgets are pruned at boot and every minute.
Ended sessions disappear when their last widget is removed.

## 6. Island behaviour

- Rest: 128 × 52 preview without agents, or a bar up to 600 px wide with planets
  split around Orbi; optionally a 104 × 28 minimized tab. Hover opens neither.
  Click preview to expand; click tab to show a preview that returns to the tab
  when minimized or after a minute without the user's pointer or a click.
- Preview planets and rail glyphs open agent tabs. Home is an orrery; clicking
  a planet dives into its subagent system, recursively. Breadcrumbs, empty
  stage, wheel or the Home rail glyph zoom out. Flights last 1.1 s; reduced
  motion uses a 260 ms crossfade. The caption opens the owning agent's tab.
- Planet appearance comes from its **session id** alone; agents cannot set
  it, and an `avatar` they send is ignored. Moons always represent reported subagents. They launch,
  orbit, flash green/red when done and fall back into the parent. Mini planets
  show at most three within their status ring; the agent view's timeline shows
  each reported subagent running and ending, with its time and parent.
- The expanded island is 600 px wide, with a 44 px rail. Agent tabs show task,
  status, the first progress widget's bar, question card and a timeline. The
  island keeps each version of a widget that says something new (a progress
  bar moving alone updates the last one) for two hours; each becomes an event,
  and a widget's newest event shows it in full: steps as a checklist, brief
  items one per line, all text wrapping.
  Entries have no individual close buttons. Closing the request hides its updates
  under `orbi.closed-requests` until the task changes, even across Orbi restarts;
  it leaves the agent running. Ended sessions can be forgotten from the rail.
- Working planets show progress or typing dots. Finished planets show green,
  errors red with `!`, questions amber with `?` and repeated hops.
- An island the user opened or entered stays open when the pointer leaves; × and Escape collapse it. One that a notice or question opened by itself, never entered, returns to rest when its pin ends.
  ^ minimizes and saves the rest preference. Pause hides it. Reopening resumes
  the previous view, choosing the first agent on the first open.
- `clarify` opens and pins the agent tab, except while resting minimized:
  there it only pulses and sounds. The looping `sonar` continues until answer,
  skip, timeout, request close or pause. A single choice without free text answers
  in one click; text entry temporarily enables window focus.
- `notify` opens the agent tab for four seconds with its message.
  Success makes Orbi happy; warning surprises it. Paused or closed requests
  suppress notifications.
- Orbi reacts to pokes, ring flicks and scrolls; a hover makes it blink. Idle Orbi sleeps after 90 s without activity and no working/thinking
  agents. Hidden island animation and inactive view animation stop.
  See [the website](https://orbi.chrystalesia.com/) for a preview of the island.

## 7. Install (`orbi-mcp install`)

`orbi-mcp install|uninstall <auto|claude|codex|gemini|agy|all>… [--yes] [--instructions] [--dry-run]` parses and
merges config files, prints diffs, takes dated backups of existing files and
writes only after confirmation. Invalid files and foreign entries are preserved.
Writes detect files changed since the diff. JSON preserves key order, BOM,
line endings and trailing newline; TOML preserves comments and formatting.

Paths below are relative to the user's home (`%USERPROFILE%` or `$HOME`).
`ORBI_MCP_HOME` redirects config lookup for rehearsals; it does not redirect
the bridge copy. `CODEX_HOME`, when absolute, selects Codex's config and instruction
directory; a rehearsal using `ORBI_MCP_HOME` ignores it.

`auto` selects existing agent profiles. `--dry-run` prints diffs without writing
or making backups. All selected files are planned before any are written, so a
parse error or conflicting entry aborts preflight without a partial install.

`--instructions` also merges the embedded `windows/mcp/agent-instructions.md`
block into Claude's global `CLAUDE.md`, Codex's global `AGENTS.md` (or nonempty
`AGENTS.override.md`), Gemini/agy's global `GEMINI.md`, and an existing shared
`~/.agents/AGENTS.md`. Only the `orbi:start` / `orbi:end` block is replaced or
removed. Duplicate or incomplete markers are errors. Repeated installs are
idempotent; instructions use the same preview, backup and write checks as config.
`uninstall all --instructions` removes the shared block too; a partial client
uninstall leaves it for the remaining clients.

| Agent | MCP server | Guard hooks: start / tool / stop |
|---|---|---|
| Claude Code | `.claude.json`: `mcpServers.orbi`, `{type:"stdio",command}` | `.claude/settings.json`: `UserPromptSubmit` / `PostToolUse` / `Stop` |
| Codex | `.codex/config.toml`: `[mcp_servers.orbi]`, `command`, `tool_timeout_sec = 620` | `.codex/hooks.json`: `UserPromptSubmit` / `PostToolUse` / `Stop` |
| Gemini CLI | `.gemini/settings.json`: `mcpServers.orbi`, `{command}` | Same file: `BeforeAgent` / `AfterTool` / `AfterAgent` |
| agy | `.gemini/config/mcp_config.json`: `mcpServers.orbi`, `{command}` | `.gemini/config/hooks.json`: named hook `orbi`, `PreInvocation` / `PostToolUse` / `Stop` |

Grok has no install of its own: by default it reads Claude Code's MCP servers
(`.claude.json`) and hooks (`.claude/settings.json`), so `install claude` covers
it. The guard recognises Grok's camelCase payload (`sessionId`, `toolName`,
`toolInput`, `stopHookActive`) whatever agent the command names.

The command points to `%LOCALAPPDATA%\Orbi\bin\orbi-mcp.exe`,
`~/.local/share/orbi/bin/orbi-mcp` on Linux (honoring `XDG_DATA_HOME`), or
`~/Library/Application Support/Orbi/bin/orbi-mcp` on macOS.
The app refreshes the copy at launch, moving a locked old binary aside as
`orbi-mcp.old-<ms>.exe` and trying to delete it on a later launch.
Codex needs new hooks trusted in `/hooks`; all agents need a fresh session.

Install migrates owned legacy Planet Bar server entries and guard hooks:
a previous server key or agy named hook is removed only when its command
identifies the legacy bridge. Legacy guard handlers are removed across hook
events, preserving other handlers in mixed groups. Uninstall removes both
generations' owned entries; foreign legacy entries remain.

### Guard (`orbi-mcp guard <agent> <start|tool|stop>`)

State lives in `<temp>/orbi-guard/<agent>-<session>`; stale files are removed
after three idle days at a later start.

- **start** adds context while Orbi runs: report on Orbi only when it helps the
  user follow the work (several steps, minutes of work, or the user waiting); a
  quick answer or small edit needs no call. A reported task takes few calls: one
  `introduce` of the repository folder (or working folder outside a repository,
  never home/drive root) carrying `state: working` and, with several steps, the
  progress widget in `widgets`; updates only on real changes, never per tool
  call, with `list` or `stat` riding along; and one `set_status` to finish with
  a `brief` in `widgets` and out-of-date ids in `remove`; clarify for questions. Once introduced, it asks for `introduce`
  only on a new task. agy's `PreInvocation` reminds once per turn.
- **tool** records Orbi calls and the state `set_status` or `introduce` set, or
  marks other tool calls as real work.
- **stop** holds a turn that reported on Orbi (an `introduce` or a state this
  turn) and did real work, but has no final finished/error status, once; work
  the agent did not report is never held:
  `block` for Claude/Codex/Grok, `deny` for Gemini, `continue` for agy. The
  next stop passes, and `stop_hook_active` is honored.
- Grok discards what a turn-start hook prints, so its reminder goes out as
  `PostToolUse` `additionalContext` on the turn's first tool call.
- While Orbi runs, every hook also writes `<temp>/orbi-guard/client-<pid>.json`,
  `{"agent","log"}`: the agent's session log, `pid` being the agent process (the
  hook's nearest ancestor that is not a shell). The log is the payload's
  `transcript_path` (Claude Code, Codex, Gemini CLI), Grok's
  `~/.grok/sessions/*/<sessionId>/chat_history.jsonl`, or agy's
  `~/.gemini/antigravity-cli/conversations/<conversationId>.db`.

The guard exits 0 within two seconds. Missing app, unreadable input or unwritable
state cannot block the agent. No-op output is empty for Claude/Codex, `{}` for
Gemini/agy (agy Stop: `{"decision":""}`). Gemini/agy hook commands check for the
binary before invoking it, emitting `{}` if absent. Installation refuses hooks
when the installed bridge is known to lack `guard`.

### Context maps

The app finds each live session's agent process from the bridge `pid` in
`hello` (its first ancestor past shells with a `client-<pid>.json`) and, while
the island is shown, reads the log named there every 3 s from where it stopped.
Claude Code, Codex and agy record the context of every model call; the growth
between two calls is split among the pieces logged in between (tool inputs and
outputs, messages, replies, thinking) by their length. Grok records no counts, so
its sizes are estimates at four characters a token. A call under 60% of the last
one is a compaction. Claude Code subagents are read from
`<transcript>/subagents/*.jsonl`; one whose prompt says
"Your Orbi subagent id is `x`" is the moon `x`. Gemini CLI has no map yet.

Pieces are grouped by MCP server, `Reading code` and `Editing code` (by file),
`Shell` (by program), `Chat` and `System & tools`. The island gets
`context-changed`, `{session_id: {used, window, estimated, steps, groups,
subagents}}`; `context_snapshot`, `context_history(session_id, sub?)` and
`context_histories` answer with the same maps and with each step's frame for
looking back through the history. Nothing is kept on disk.

