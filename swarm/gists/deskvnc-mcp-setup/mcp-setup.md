# Registering the DeskVNC MCP server with an AI client

Register `dvv`, the MCP server that ships inside DeskVNCViewer, with the AI agent
on your machine. `dvv` lets the agent open one of your saved machines, observe
the screen, and send clicks and keystrokes over the ordinary VNC, RDP or SSH
connection, so nothing is installed on the remote target. The README documents
the exact one-liner for macOS Claude Code, and a built-in helper, `dvv doctor`,
that prints the right line for any other client.

## macOS Claude Code, the verified one-liner (from the README)

```sh
claude mcp add deskvnc -- /Applications/DeskVNCViewer.app/Contents/MacOS/dvv mcp --stdio
```

The HTTP transport is verified on macOS too. Set `DVV_MCP_TOKEN` first so the
token survives a restart, then:

```sh
claude mcp add --scope user --transport http deskvnc http://127.0.0.1:7333/mcp --header "Authorization: Bearer $DVV_MCP_TOKEN"
```

Claude Code and OpenCode are verified over both stdio and HTTP. Anything else
uses `dvv doctor`, which prints the exact line for the client it can see on
your PATH, so the rest of this card stays at the level of "here is the shape"
for the unverified entries.

## Linux, swap the binary path

`dvv` lives at `/usr/bin/dvv` on the standard Linux build (a `deb` with a
system install). The Claude Code command becomes:

```sh
claude mcp add deskvnc -- /usr/bin/dvv mcp --stdio
```

The AppImage runs from a folder that disappears on quit, so setup copies `dvv`
into `~/.local/share/DeskVNCViewer/bin/dvv` and points agents there. After the
copy, the same shape holds, just with that path:

```sh
claude mcp add deskvnc -- $HOME/.local/share/DeskVNCViewer/bin/dvv mcp --stdio
```

HTTP transport on Linux uses the same URL and bearer header, with `localhost`
in place of the macOS path:

```sh
claude mcp add --scope user --transport http deskvnc http://127.0.0.1:7333/mcp --header "Authorization: Bearer $DVV_MCP_TOKEN"
```

## Windows, swap the binary path and add `.exe`

`dvv.exe` lives beside the app, in `%LOCALAPPDATA%\DeskVNCViewer` for a
per-user install or in `C:\Program Files\DeskVNCViewer` for the all-users
install. PowerShell:

```powershell
claude mcp add deskvnc -- "$env:LOCALAPPDATA\DeskVNCViewer\dvv.exe" mcp --stdio
```

For the all-users install, substitute `"C:\Program Files\DeskVNCViewer\dvv.exe"`.
HTTP transport on Windows uses the same loopback URL and bearer header as the
other platforms:

```powershell
claude mcp add --scope user --transport http deskvnc http://127.0.0.1:7333/mcp --header "Authorization: Bearer $env:DVV_MCP_TOKEN"
```

## OpenCode, both transports verified

OpenCode reads `opencode.json` at the project root or
`~/.config/opencode/opencode.json`. With a system install on macOS:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "deskvnc": {
      "type": "local",
      "command": ["/Applications/DeskVNCViewer.app/Contents/MacOS/dvv", "mcp", "--stdio"],
      "enabled": true
    },
    "deskvnc-http": {
      "type": "remote",
      "url": "http://127.0.0.1:7333/mcp",
      "headers": { "Authorization": "Bearer <token>" },
      "enabled": true
    }
  }
}
```

On Linux and Windows, swap the `command` path to `/usr/bin/dvv` or
`%LOCALAPPDATA%\DeskVNCViewer\dvv.exe`, and substitute Unix-style backslashes
for the platform's path separator. The HTTP entry is identical across
platforms.

## Other clients, covered and unverified

Codex CLI, Cursor, Windsurf, VS Code, Gemini CLI and plain Python agents
follow the same two shapes (`command` plus `args` for stdio, `url` plus a
bearer `headers` block for HTTP), and the integration notes in
`docs/AGENTS.md` spell each one out. Two small differences to know:

* VS Code uses the key `servers` rather than `mcpServers`, and takes
  `"type": "stdio"` on the entry.
* Gemini CLI takes `httpUrl` in place of `url` for the HTTP transport.

Everything outside Claude Code and OpenCode is covered by those shapes and
worth a glance at the current docs of that client. The fastest way to skip
the question entirely is to run `dvv doctor` once the app is installed; it
prints the exact line for any client it can find on your machine.

## A note on what's verified

Verified end to end by the project's own workflow against the shipped build:
Claude Code on both transports, OpenCode on both transports, and a long-lived
peer used to drive a saved machine. The Linux `deb`, the AppImage, and the
AppImage copy to `~/.local/share/DeskVNCViewer/bin/dvv` are checked by the
same workflow. The Windows and Linux paths above are the documented install
locations written in `docs/AGENTS.md`, not freshly exercised registrations
on this card. Run `dvv doctor` for the line that matches the specific client
and platform you have.

## Links

* Project: <https://github.com/psmux/DeskVNC>
* Integration notes for every client: <https://github.com/psmux/DeskVNC/blob/main/docs/AGENTS.md>
* Page: <https://psmux.github.io/deskvnc/>
