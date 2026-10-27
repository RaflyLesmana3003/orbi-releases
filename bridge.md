# The agent bridge

[Website](https://orbi.chrystalesia.com/) ? [Install](install.md) ? [Tool contract](tools.md)

`orbi-mcp` connects an AI coding agent to its planet on Orbi. Each session runs one
bridge process and speaks MCP over stdio. The bridge exposes eight tools, then
forwards validated JSON lines to the app. See the [contract](tools.md)
for parameters, limits and replies.

At launch, the app copies it to a stable location:

| Platform | Installed bridge |
|---|---|
| Windows | `%LOCALAPPDATA%\Orbi\bin\orbi-mcp.exe` |
| macOS | `~/Library/Application Support/Orbi/bin/orbi-mcp` |
| Linux | `~/.local/share/orbi/bin/orbi-mcp`, under `XDG_DATA_HOME` when set |

Agent configs point to that copy so they survive app updates.

## Commands

| Command | Behaviour |
|---|---|
| `orbi-mcp` | Serve MCP over stdio; there is no `serve` subcommand |
| `orbi-mcp install <auto\|claude\|codex\|gemini\|agy\|all>… [--yes] [--instructions] [--dry-run]` | Register the `orbi` MCP server and guard hooks; optionally manage global instructions |
| `orbi-mcp uninstall <auto\|claude\|codex\|gemini\|agy\|all>… [--yes] [--instructions] [--dry-run]` | Remove owned server entries, hooks, and optionally instructions |
| `orbi-mcp guard <claude\|codex\|gemini\|agy> <start\|tool\|stop>` | Read hook JSON from stdin and emit the agent's hook response |
| `orbi-mcp version` | Print the bridge version; `--version` and `-V` also work |
| `orbi-mcp help` | Print usage; `--help` and `-h` also work |

Install and uninstall parse and merge each file, show its diff, then write only
after an interactive `y` or `--yes` (`-y`). Existing files receive dated backups.
Invalid files and conflicting foreign entries are left alone. JSON preserves key
order, BOM, line endings and trailing newline; TOML preserves comments and formatting.
The file is reread before writing to detect edits made during confirmation.
All selected files are checked before any writes; a later write failure stops
the operation and reports the error. Backups do not overwrite earlier backups.

### Complete setup

The recommended [one-line installation guide](install.md) downloads
Orbi, starts it, then runs `install auto --instructions`. `auto` selects existing
profiles rather than creating configurations for agents you do not use. Pass
`--dry-run` to inspect every change without writing, or `--yes` to apply changes
you have already approved.

`--instructions` manages only the block between `<!-- orbi:start -->` and
`<!-- orbi:end -->`. Its [source](agent-instructions.md) covers progress, questions,
widgets, finishing tasks, and authorized subagents. It updates existing blocks
on repeat installs and rejects incomplete or duplicate markers.

| Client | Instruction file |
|---|---|
| Claude Code | `~/.claude/CLAUDE.md` |
| Codex | `<Codex home>/AGENTS.md`, or a nonempty `AGENTS.override.md` |
| Gemini CLI and agy | `~/.gemini/GEMINI.md` |
| Shared rules | `~/.agents/AGENTS.md`, if it exists |

`uninstall all --instructions` removes these blocks as well as MCP and hooks.
A partial uninstall keeps the shared `.agents/AGENTS.md` block. Without
`--instructions`, install and uninstall leave all instruction files alone.
Gemini's shared rules also remain when the other Gemini/agy client is connected.

### Agent configuration

Paths below are relative to the user's home directory. Codex uses `CODEX_HOME`
when it is set to an absolute path, otherwise `~/.codex`. Its instruction-file
selection follows [Codex's global instruction precedence](https://learn.chatgpt.com/docs/agent-configuration/agents-md).

| Agent | MCP entry | Guard hooks |
|---|---|---|
| Claude Code | `.claude.json`: `mcpServers.orbi`, with `type: "stdio"` and `command` | `.claude/settings.json`: `UserPromptSubmit`, `PostToolUse`, `Stop` |
| Codex | `.codex/config.toml`: `[mcp_servers.orbi]`, `command`, `tool_timeout_sec = 620` | `.codex/hooks.json`: `UserPromptSubmit`, `PostToolUse`, `Stop` |
| Gemini CLI | `.gemini/settings.json`: `mcpServers.orbi`, with `command` | Same file: `BeforeAgent`, `AfterTool`, `AfterAgent` |
| agy | `.gemini/config/mcp_config.json`: `mcpServers.orbi`, with `command` | `.gemini/config/hooks.json`: named hook `orbi`, with `PreInvocation`, `PostToolUse`, `Stop` |

The command is the installed bridge path above. Codex needs new hooks trusted in
`/hooks`; start a new agent session after installation. Its 620-second tool timeout
covers `clarify`'s maximum 600-second wait and the bridge's reply grace period.

Install removes a previous Planet Bar MCP entry and guard hooks only when their
commands identify the legacy bridge. Uninstall removes owned entries from both
generations. Other servers and hook handlers remain, including foreign entries
under the old key. A bridge known to lack `guard` is refused for hook installation.

The guard reminds agents to report on Orbi only when it helps the user follow the
work, in few calls, and to ask questions through `clarify`. It holds a turn once
if it reported on Orbi but ends without a `finished` or `error` state, from
`set_status` or `introduce`; work the agent did not report is never held. While
Orbi runs, it also leaves the path of the agent's session log in
`<temp>/orbi-guard/client-<pid>.json` for the app's context map. With Orbi closed, it does nothing;
it always exits 0 within two seconds. Grok runs Claude Code's hooks with its own
payload, which the guard recognises. See [contract §7](tools.md#7-install-orbi-mcp-install).

`ORBI_MCP_HOME` redirects config lookup to a copied home directory and ignores
`CODEX_HOME` so a rehearsal cannot touch the real profile. It does not change the
installed bridge path:

```powershell
$env:ORBI_MCP_HOME = "C:\tmp\home"
.\target\release\orbi-mcp.exe install claude
```

## Behaviour when Orbi is closed

The Windows endpoint is `\\.\pipe\orbi-agents-<sid>`. macOS uses
`~/Library/Application Support/Orbi/orbi-agents.sock`; Linux uses
`$XDG_RUNTIME_DIR/orbi-agents.sock` (fallback `/run/user/<uid>/orbi-agents.sock`).
The app includes listeners for all three platforms. Linux runtime validation is
still pending.

Connection starts on the first tool call, with a three-second initial connection
budget. Without a connection, tools return `{"ok":false,"error":"orbi-not-running"}`.
The bridge does not retry in the background: an agent idle while Orbi restarts
stays off the island until its next tool call, which reconnects and replays
`hello`, the last `introduce`, widgets and running subagents. Ordinary writes and replies
have three-second timeouts; a stalled reply returns `timeout`. `clarify` waits for
the user separately. Diagnostics go to stderr; stdout belongs to MCP.

