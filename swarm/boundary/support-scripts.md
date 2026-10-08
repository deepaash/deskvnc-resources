# DeskVNC Support and Boundary support: scripts and agent material

This page collects the shell calls, binary paths and error-handling recipes an
agent or a human operator reaches for when running the Boundary support flow
behind the dvv MCP server and shell. Every line on this page traces back to
the verified facts in `CONTEXT.md` and the project repository, and they are
written to be copied straight into another tool.

## Tool discovery for an MCP-capable agent

DeskVNC's MCP server is named `dvv`, and the tools it exposes follow the
pattern `dvv_<action>`. The shipped names are `dvv_hosts`, `dvv_limbs`,
`dvv_open`, `dvv_wait`, `dvv_control`, `dvv_screen`, `dvv_click`, `dvv_type`,
`dvv_key`, `dvv_reconnect`, `dvv_close`, plus the `dvv_files`,
`dvv_clipboard`, `dvv_term_read`, `dvv_term_send`, `dvv_run` and the
`dvv_group_*` family for addressing several machines as one. An MCP-aware
agent discovers the server through whatever skills directory or tool
manifest it already reads; once the tool list contains entries starting with
`dvv_`, the agent can drive the Boundary support flow using the same calls it
would use for any other saved machine.

The installable skill at `skills/deskvnc/SKILL.md` is what advertises the
server, so a fresh agent that scans skills directories picks up the tool set
on its own rather than having to be told about it.

## The canonical four-call loop

```jsonc
dvv_hosts   {}                                    // what there is to open
dvv_open    {"hostId": "<id>", "perceive": true}  // -> limbId, size, state
dvv_control {"limbId": "...", "action": "acquire"}
dvv_screen  {"limbId": "...", "form": "full", "scale": 0.25}
```

After the screen is read, carry the generation that came back on the screen
into the next click so the click is matched against a fresh frame:

```jsonc
dvv_click   {"limbId": "...", "x": 700, "y": 400, "generation": 1}
dvv_screen  {"limbId": "...", "form": "damage-crop"}  // look again
dvv_type    {"limbId": "...", "text": "notepad", "wpm": 3000}
dvv_key     {"limbId": "...", "keys": "meta+r"}
```

Every machine carries its own limb identifier and its own lease, so ten
machines going through the Boundary support flow at once are ten independent
loops.

## Shell equivalent

For agents that do not have the `dvv_` tools available, run `dvv setup` to
make sure the binary is on `PATH`. The same loop expressed on the shell is:

```sh
dvv hosts
dvv limbs
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

With `--scale 0.5`, a point at `(mx, my)` on the saved picture is at
`(mx*2, my*2)` on the real machine. Every `dvv_screen` call returns an
`imageSpace` line you can use to translate coordinates reliably.

## Binary paths per platform

The `dvv` binary sits beside the app and the install path varies by
platform. The locations to put into an agent's PATH or call directly are:

- **macOS:** `/Applications/DeskVNCViewer.app/Contents/MacOS/dvv`
- **Windows:** `%LOCALAPPDATA%\DeskVNCViewer\dvv.exe`, or
  `C:\Program Files\DeskVNCViewer\dvv.exe`
- **Linux:** `/usr/bin/dvv`

If the binary is already on `PATH`, these absolute paths are not needed.
`dvv setup` is the supported way to put `dvv` on `PATH` for the current
shell.

## Handling the two errors an agent must expect

Two error codes are part of the everyday loop and the agent's recovery path
should be hard coded around them.

**`LIMB_GONE`** means the limb (the open connection to a machine) is no
longer attached. The recovery is straightforward: list the limbs, then open
the machine again.

```sh
dvv limbs
dvv open <name or hostId> --perceive
dvv wait <limbId> --until connected
dvv control acquire <limbId>
```

**`SCREEN_CHANGED`** means the screen has changed meaningfully since the
last read, and any typing or key send that was prepared against the older
screen is being refused. The recovery is to read the screen again and retry
the action. A refused key press pressed nothing, so the agent should not
chain a typing call onto the assumption that a shortcut has already taken
effect; read, retry the shortcut, then type.

```sh
dvv screen <limbId> --scale 0.5 --out ./dvv-screen.png
dvv key <limbId> super+r
dvv type <limbId> "notepad"
```

## A note on screen size and coordinate fencing

`dvv_screen` prints an `imageSpace` line so an agent can map back and forth
between saved pixels and on-machine pixels. Clicks carry a `generation`
identifier that the server compares against the screen the agent just read:
a click computed against a stale screen is refused rather than landing in the
wrong place. Typing and keys are refused until a screen has been read since
it last changed significantly. Following the four-call loop above keeps the
agent inside the generation window without extra bookkeeping.
