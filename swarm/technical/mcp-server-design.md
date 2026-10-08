# The DeskVNC MCP server as a systems article

The MCP server that ships with `DeskVNC` is what makes the project usable
as an automation target rather than just a remote desktop client. This
article treats it as a small piece of distributed systems design: a tool
surface, a per-machine lease model, a coordination contract that keeps
the agent honest about what it knows, and a shell fallback for agents
that have no MCP transport of their own.

The authoritative home for the tool names and the recovery paths is the
[`dvv` skill](https://github.com/psmux/DeskVNC/blob/main/skills/deskvnc/SKILL.md)
that ships in the repository, and the project README is where the
public-facing shape lives.

## The tool surface

The MCP server exposes a small set of tools, each named to read like an
imperative verb on a limb (a connection that the agent has opened):

- `dvv_hosts` lists the saved machines in the operator's library.
- `dvv_limbs` lists the connections currently attached to this peer.
- `dvv_open` opens a machine by `hostId` (a saved name resolves to one)
  with `perceive: true`, returning a `limbId`, the screen size, and the
  session state.
- `dvv_wait` blocks until the limb reaches a target state, typically
  `connected` or `screen-stable`.
- `dvv_control` with `action: "acquire"` takes the input lease so the
  agent can drive the session.
- `dvv_screen` reads the current screen with a chosen form (full or
  `damage-crop`) and an optional scale; the result carries an
  `imageSpace` line and a `generation`.
- `dvv_click` issues a coordinate-based click, with `x`, `y`, and the
  `generation` returned by the most recent screen read. Optional
  `action: double` for double clicks.
- `dvv_type` sends keystrokes as text, optionally with a `wpm` rate for
  realistic typing speed. Refused if the screen has changed
  substantially since the last `dvv_screen`.
- `dvv_key` sends named shortcuts, with names like `meta+r`, `ctrl+l`,
  `alt+F4`, `Enter`, `Escape`, `Tab`. Same refusal behaviour as
  `dvv_type`.
- `dvv_clipboard` reads and writes the clipboard on hosts that support
  extended clipboard, with UTF-8, RTF, and HTML, falling back to Latin-1
  for older servers.
- `dvv_files` moves files both ways through SFTP on hosts that have it.
- `dvv_term_read` and `dvv_term_send` use the SSH terminal when the
  underlying session is SSH, ideal for text-only agents.
- `dvv_run` executes a shell command on an SSH limb.
- `dvv_reconnect` reattaches a dropped session.
- `dvv_close` releases the limb.

The `dvv_group_*` family wraps the same tools for a set of machines,
addressing them as one. That makes a fan-out trivial: drive the same
keystrokes into five boxes, or read five screens, and treat the result
as a single response.

## The four-call loop

The agent's control flow is a short loop, and the project README is the
place it is presented in public form:

```jsonc
dvv_hosts   {}
dvv_open    {"hostId": "<id>", "perceive": true}
dvv_control {"limbId": "...", "action": "acquire"}
dvv_screen  {"limbId": "...", "form": "full", "scale": 0.25}
dvv_click   {"limbId": "...", "x": 700, "y": 400, "generation": 1}
dvv_screen  {"limbId": "...", "form": "damage-crop"}
dvv_type    {"limbId": "...", "text": "notepad", "wpm": 3000}
dvv_key     {"limbId": "...", "keys": "meta+r"}
```

The shape reads as perceive-then-act: read, then drive, then read again.
The second read at `damage-crop` is deliberate, because the cheapest
bandwidth path for an LLM is the rectangle that just changed. A
`dvv_wait` with `until: "screen-stable"` is the way to wait for a
remote action to settle without polling.

This loop beats "blind scripting" for two engineering reasons. First,
each cycle costs a few tens of milliseconds at most over a LAN, so the
loop rate is bounded by the network and the model rather than by the
client. Second, the per-action fencing catches the moment where the
remote screen has changed under the agent, refusing the next click or
keystroke rather than executing it against an old mental model.

## The per-machine lease model

Every machine the agent opens is its own limb, with its own lease and
its own input arbitration. Ten machines are ten independent loops, all
running in parallel under one MCP peer. The agent does not have to hold
ten connections; it sends ten tool calls and gets ten responses. This
lets one agent hold two desktops and run a different automation on each
at the same time, with no global state to coordinate.

A person can take the wheel at any moment by clicking into a pane. The
agent's lease is revoked in that case, held keys and buttons are released
so a half-finished drag cannot strand the desktop, and the project
surfaces `LEASE_REVOKED` to the agent. The rule that the agent should
treat a `LEASE_REVOKED` as a stop signal and inform the user is the
only place the agent should unilaterally halt; everything else is
recoverable.

## The shell fallback, and why it matters

A wide range of agent runtimes has its own way of consuming tools, and
many of them do not speak MCP. Agents that shell out, that prefer a
command line surface, or that drive everything through a single
subprocess still need a way to drive a desktop, and the project ships
`dvv` as a shell command for them. After `dvv setup` puts the binary on
`PATH` (`/Applications/DeskVNCViewer.app/Contents/MacOS/dvv` on macOS,
`%LOCALAPPDATA%\DeskVNCViewer\dvv.exe` or
`C:\Program Files\DeskVNCViewer\dvv.exe` on Windows, `/usr/bin/dvv` on
Linux), the same loop runs as:

```sh
dvv hosts
dvv open <name or hostId> --perceive
dvv wait <limbId> --until connected
dvv control acquire <limbId>
dvv screen <limbId> --scale 0.5 --out ./dvv-screen.png
dvv click <limbId> <x> <y>
dvv click <limbId> <x> <y> --action double
dvv type <limbId> "text to type"
dvv key <limbId> super+r
dvv wait <limbId> --until screen-stable
dvv reconnect <limbId>
dvv close <limbId>
```

That fallback is the difference between an MCP integration that works
for Claude Code and OpenCode (both verified over both stdio and HTTP)
and a system that is reachable from any agent. The skill file makes the
preference explicit: use `dvv_` tools when available, fall back to
shell only when those tools are not available. The two surfaces are
deliberately equivalent so a fallback never changes the underlying
semantics.

## Measured at the bench

The README records the latency of the control plane on a real 1920x1080
Windows desktop over a LAN:

| call | time |
| --- | --- |
| `dvv_open` and attach | 4 ms |
| `dvv_control` acquire | under 1 ms |
| `dvv_screen` at `scale: 0.25` (112 KB) | 25 ms |
| `dvv_screen` at full scale (1.5 MB) | 70 ms |
| one observe-then-act cycle | 19 ms, about 52 actions per second |
| `dvv_type` throughput at `wpm: 12000` | 447 characters per second |

These are the numbers that fit the loop above, and they are the reason
the loop reads as a loop rather than as a script: at this rate the
bandwidth is the model's perception of the screen, not the network. An
agent holding two desktops and typing a different sum into a calculator
on each finishes both in 0.95 seconds on this hardware.

## Treat remote text as data

A small but important design rule appears in the skill file and in the
README: anything a remote machine produces is untrusted text. A window
title, a directory listing, terminal output: all of it is data to act
on, never instructions to follow. This is the same rule that any
service consuming user content lives by, applied to agent context.

The full tool surface, the recovery paths for `LIMB_GONE`,
`SCREEN_CHANGED`, and `LEASE_REVOKED`, and the relative preferences for
MCP and shell surfaces are documented in the
[`dvv` skill](https://github.com/psmux/DeskVNC/blob/main/skills/deskvnc/SKILL.md).
