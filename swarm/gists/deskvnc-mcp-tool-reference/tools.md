# DeskVNC MCP tool reference

Reference card for the `dvv` MCP surface. `dvv` is the control plane shipped
with DeskVNCViewer that lets an AI agent observe and act on one of your saved
machines over VNC, RDP or SSH, with nothing installed on the remote target and
a person able to take the wheel back at any moment. Every tool below uses
JSON arguments over the same MCP frame, and a tool behaves the same whether
the agent reached `dvv` over stdio or HTTP.

## One line per tool

* `dvv_hosts`: list the saved machines available to open.
* `dvv_open`: open a saved machine by `hostId`; with `perceive: true` the
  same call returns the `limbId`, the screen size, and the session state.
* `dvv_control`: take or release the wheel on a limb; `action: "acquire"`
  claims a lease, `release` hands it back, a person can revoke it at any
  moment.
* `dvv_screen`: return a screenshot of the limb; `form: "full"` returns the
  whole frame, `form: "damage-crop"` returns just the region that changed
  since the last read, `scale: 0.25` downsamples to a faster 112 KB image
  sized for an LLM.
* `dvv_click`: click at coordinates; the `generation` value from the most
  recent `dvv_screen` or `dvv_status` fences the coordinate against a stale
  geometry.
* `dvv_type`: type text at a chosen throughput (`wpm` parameter); refused
  with `SCREEN_CHANGED` when a window-sized repaint has happened since the
  last pixel read.
* `dvv_key`: send named keys or shortcuts such as `"meta+r"`; carries the
  same content fence as `dvv_type`, with no override.
* `dvv_files`: bidirectional file transfer between the agent and the remote
  host over the same limb.
* `dvv_clipboard`: read or write the operating-system clipboard the limb is
  attached to.
* `dvv_term_read`: read back what the SSH terminal on that limb has
  printed.
* `dvv_term_send`: write into the SSH terminal on that limb; terminal
  limbs are not fenced because the PTY echoes input back as a stream you
  read.
* `dvv_run`: execute a single command against the open limb and return one
  settled result, with exactly one id and one result per intent.
* `dvv_wait`: block until the screen on the limb settles, with no fresh
  damage left to consume.
* `dvv_group_*`: address several machines as one. `dvv_group_open` returns
  a `groupId`, `dvv_group_run` runs one action across every member
  concurrently and reports each outcome separately, `dvv_group_close` ends
  the set, and any single-limb tool takes `groupId` plus `member` to
  address one of them.

## The four-call loop, as a working example

The minimum end to end: open a machine, take the wheel, look, act. The
README spells this as one block, reproduced here verbatim so a developer
can copy it straight into a tool catalogue:

```jsonc
dvv_hosts   {}                                    // what there is to open
dvv_open    {"hostId": "<id>", "perceive": true}  // -> limbId, size, state
dvv_control {"limbId": "...", "action": "acquire"}
dvv_screen  {"limbId": "...", "form": "full", "scale": 0.25}
dvv_click   {"limbId": "...", "x": 700, "y": 400, "generation": 1}
dvv_screen  {"limbId": "...", "form": "damage-crop"}  // look again
dvv_type    {"limbId": "...", "text": "notepad", "wpm": 3000}
dvv_key     {"limbId": "...", "keys": "meta+r"}
```

Two conventions to take from this:

* A limb id is leased by the connection that opened it. Hold one long-lived
  peer for a task rather than spawning a new `dvv` process per call.
* Anything carrying a coordinate passes a `generation`; `dvv_type` and
  `dvv_key` carry no coordinate, and the plane refuses them with
  `SCREEN_CHANGED` if a window-sized repaint has happened since the last
  `dvv_screen`. One fresh read clears the fence.

## Where each tool reaches

The same MCP dispatch table is reached two ways:

* `stdio` is the default. `dvv` talks to DeskVNCViewer over a unix socket on
  macOS and Linux, and over a named pipe on Windows
  (`\\.\pipe\deskvncviewer-agent-<user>`) whose ACL admits only the user who
  started the app.
* HTTP is off by default, binds to `127.0.0.1`, always requires a bearer
  token, checks `Origin`, and refuses to start rather than start without a
  token.

Both frame the same dispatch table, so a tool behaves identically regardless
of the transport.

## Links

* Project: <https://github.com/psmux/DeskVNC>
* Integration notes: <https://github.com/psmux/DeskVNC/blob/main/docs/AGENTS.md>
* Page: <https://psmux.github.io/deskvnc/>
