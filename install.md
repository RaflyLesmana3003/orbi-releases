# Install Orbi

The user has asked you, their AI agent, to install Orbi, connect their agents, and add its global instructions. Carry out the setup on their desktop computer. Orbi is a desktop island; its local `orbi-mcp` bridge supplies progress, questions, widgets, and subagent reporting. No account, API key, Node, or Rust installation is needed to use a release.

Explain the app location and agent profiles you will change. This request authorizes the setup described here; follow your host's approval rules and ask only for permission or information you still need. Preserve existing settings and instructions. Do not install unrelated agents or change their approval policy.

## Choose the release

Detect the desktop OS and CPU architecture. These are the published targets:

| Desktop | Release tag | Required assets |
|---|---|---|
| Windows x64 | `windows-latest` | `orbi.exe`, `orbi-mcp.exe` |
| macOS Apple Silicon | `macos-latest` | `Orbi-macOS-aarch64.dmg` |
| Linux x86_64 | `linux-latest` | `Orbi-Linux-x86_64.AppImage` |

Read `https://api.github.com/repos/RaflyLesmana3003/orbi-releases/releases/tags/<tag>`. For each required asset, use its `browser_download_url` and `digest` (`sha256:…`). Download into a temporary directory and verify its SHA-256 before replacing or running anything. Stop on a missing asset, digest, download failure, or mismatch. Each OS has its own release; do not guess filenames or use the generic latest release. No prebuilt download is currently published for other architectures.

Use a native local agent session. A remote container or WSL session cannot use the desktop's bridge through its own local socket.

## Install and launch

### Windows

Use PowerShell. This downloads and checks both portable executables before updating the per-user installation. It adds Orbi to the Start menu.

```powershell
$ErrorActionPreference = 'Stop'
$orbiDir = Join-Path $env:LOCALAPPDATA 'Orbi'
$orbiStage = Join-Path ([IO.Path]::GetTempPath()) ('orbi-install-' + [guid]::NewGuid())
New-Item -ItemType Directory -Path $orbiStage | Out-Null
$release = Invoke-RestMethod 'https://api.github.com/repos/RaflyLesmana3003/orbi-releases/releases/tags/windows-latest'
foreach ($file in 'orbi.exe', 'orbi-mcp.exe') {
    $assets = @($release.assets | Where-Object name -eq $file)
    if ($assets.Count -ne 1 -or $assets[0].digest -notmatch '^sha256:([a-fA-F0-9]{64})$') {
        throw "Missing release asset or SHA-256 digest: $file"
    }
    $expected = $Matches[1]
    $download = Join-Path $orbiStage $file
    Invoke-WebRequest -UseBasicParsing -Uri $assets[0].browser_download_url -OutFile $download
    if ((Get-FileHash -LiteralPath $download -Algorithm SHA256).Hash -ne $expected) {
        throw "SHA-256 mismatch: $file. Nothing installed."
    }
}
New-Item -ItemType Directory -Force -Path $orbiDir | Out-Null
$orbiExe = Join-Path $orbiDir 'orbi.exe'
Get-Process orbi -ErrorAction SilentlyContinue | Where-Object { $_.Path -eq $orbiExe } | Stop-Process
$stamp = Get-Date -Format 'yyyyMMdd-HHmmss-fff'
foreach ($file in 'orbi.exe', 'orbi-mcp.exe') {
    $dest = Join-Path $orbiDir $file
    if (Test-Path -LiteralPath $dest) { Move-Item -LiteralPath $dest -Destination "$dest.bak-$stamp" }
    Copy-Item -LiteralPath (Join-Path $orbiStage $file) -Destination $dest
}
$shortcutPath = Join-Path $env:APPDATA 'Microsoft\Windows\Start Menu\Programs\Orbi.lnk'
$shortcut = (New-Object -ComObject WScript.Shell).CreateShortcut($shortcutPath)
$shortcut.TargetPath = $orbiExe
$shortcut.Save()
Start-Process -FilePath $orbiExe
```

If WebView2 is missing, install Microsoft's WebView2 Evergreen Runtime using its official installer. If Windows blocks an unsigned download, report the exact message and let the user approve it through Windows. Do not disable protection or add antivirus exclusions.

### macOS

Download the DMG from the release metadata into a temporary directory. Compare `shasum -a 256 <download>` with the asset's digest. Then mount it with `hdiutil attach -nobrowse -mountpoint <temporary-mount> <download>`.

Install into `~/Applications/Orbi.app`, which needs no administrator access. If Orbi is running, quit it before replacing it. Move an existing app to a dated backup, then use `ditto <temporary-mount>/Orbi.app ~/Applications/Orbi.app`. Detach the image with `hdiutil detach <temporary-mount>` even if copying fails, and launch with `open ~/Applications/Orbi.app`.

The app is not notarized. If macOS asks for permission to open it, have the user approve Orbi in Privacy & Security. Do not remove quarantine attributes or disable Gatekeeper.

### Linux

Download the AppImage from the release metadata into a temporary directory. Compare `sha256sum <download>` with the asset's digest. Create `~/.local/bin`, preserve any existing `~/.local/bin/orbi` as a dated backup, and copy the verified download there. Run `chmod +x ~/.local/bin/orbi`, then launch it in the background with `nohup`, saving its output to a file in that temporary directory.

If the system has no FUSE, use AppImage's `--appimage-extract-and-run` option. Report missing system libraries from the launch log; do not claim the app started just because a background process was launched. Linux runtime validation is still pending.

## Connect the bridge and global instructions

Orbi copies its bridge into place at launch. Wait up to 30 seconds for it:

| Desktop | Installed bridge |
|---|---|
| Windows | `%LOCALAPPDATA%\Orbi\bin\orbi-mcp.exe` |
| macOS | `~/Library/Application Support/Orbi/bin/orbi-mcp` |
| Linux | `${XDG_DATA_HOME:-$HOME/.local/share}/orbi/bin/orbi-mcp` |

An unset or relative `XDG_DATA_HOME` uses `~/.local/share`. On an update, an old bridge may already exist: on Windows compare its SHA-256 with the downloaded `orbi-mcp.exe`; on macOS compare it with `Orbi.app/Contents/Resources/orbi-mcp`. Also run `version` and `help`. If the bridge did not refresh, stop and report the app's startup error. Do not point agent settings at a temporary download.

For a current bridge, preview and then apply the complete setup:

```text
<bridge> install auto --instructions --dry-run
<bridge> install auto --instructions --yes
```

On Windows, quote the executable and use PowerShell's call operator:

```powershell
& "$env:LOCALAPPDATA\Orbi\bin\orbi-mcp.exe" install auto --instructions --dry-run
if ($LASTEXITCODE -ne 0) { throw 'Orbi setup preview failed.' }
& "$env:LOCALAPPDATA\Orbi\bin\orbi-mcp.exe" install auto --instructions --yes
if ($LASTEXITCODE -ne 0) { throw 'Orbi agent setup did not complete.' }
```

Check the preview's exit status before applying. `auto` selects existing Claude Code, Codex, Gemini CLI, and agy profiles. Grok uses Claude Code's MCP settings. For an agent without a profile yet, name it explicitly instead of `auto`. Do not use `all` merely to create settings for agents the user does not use.

`CODEX_HOME` selects the active Codex profile when set to an absolute path. Otherwise Codex uses `~/.codex`. Leave the test-only `ORBI_MCP_HOME` unset. For other nonstandard profiles, inspect the agent's actual configuration before editing it; the installer covers the default locations below.

With `--instructions`, the installer registers the MCP server and guard hooks, then manages a block between `<!-- orbi:start -->` and `<!-- orbi:end -->` in:

| Agent | Global instructions |
|---|---|
| Claude Code | `~/.claude/CLAUDE.md` |
| Codex | `<Codex home>/AGENTS.md`, or its nonempty `AGENTS.override.md` |
| Gemini CLI / agy | `~/.gemini/GEMINI.md` |
| Shared agent rules | `~/.agents/AGENTS.md`, when it already exists |

Existing files receive dated backups beside them. Repeating setup updates Orbi's block instead of duplicating it. Unrelated settings and instructions remain. Invalid files, incomplete markers, and conflicting entries stop preflight before any files are written. If a write later fails, report the partial result and backup paths instead of claiming success.

### Compatibility with an older release

If `help` does not list `--instructions`, use the older bridge's `install` command with explicit client names. Detect them from the profile locations above; agy uses `~/.gemini/config`, while Gemini CLI uses `~/.gemini/settings.json`. Review its preview and confirm the writes. Do not pass new flags to an older executable.

Fetch the [Orbi instruction block](https://raw.githubusercontent.com/RaflyLesmana3003/orbi-releases/main/agent-instructions.md). Show the intended diff, back up each applicable global instruction file, then append that block or replace only its existing marked block. Preserve text outside the markers, encoding, and line endings. Stop on duplicate or incomplete markers. Check the older installer's printed paths before confirming: if it ignored `CODEX_HOME`, do not apply it to the wrong profile. Merge only the Orbi entries into the active profile using the [bridge configuration reference](https://github.com/RaflyLesmana3003/orbi-releases/blob/main/bridge.md).

## Verify and finish

- Confirm Orbi is running and the bridge's `version` command succeeds.
- Re-run the setup preview when supported. It should report no further changes.
- Reload MCP servers if the host supports it; otherwise tell the user to start a new agent session. Codex hook trust is a user action: open `/hooks` and trust Orbi's hooks when prompted. Do not edit trust records to bypass it.
- When MCP reload is available, verify all eight tools: `introduce`, `set_status`, `show_widget`, `remove_widget`, `list_widgets`, `notify`, `clarify`, `subagent`. Report a short setup task and finish it, removing its temporary widgets. Do not ask a dummy question or launch a real subagent just to test setup.
- Tell the user the app location, configured profiles, instruction files, backup paths, and any remaining restart or trust action. Until MCP reload is verified, say "configured; a new session is needed", not "connected".

The bridge and hooks enable progress, questions, widgets, subagents, and context log discovery where the client supplies it. Leave appearance, sound, launch at login, and other personal preferences as the user set them.

## Uninstall

Run `<bridge> uninstall all --instructions --dry-run`, then `<bridge> uninstall all --instructions --yes`. It removes Orbi's owned server entries, hooks, and marked instructions, including the shared block. Omitting `--instructions` leaves the rules in place. A partial client uninstall leaves the shared `.agents/AGENTS.md` block for the remaining clients.

For an older bridge, run `uninstall all --yes` and remove only the marked instruction blocks, backing up existing files first. Then quit Orbi and remove the installed app and its shortcut. Keep preferences and logs unless the user also asks to remove their data.
