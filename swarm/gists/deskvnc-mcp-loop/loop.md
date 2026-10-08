# The observe-then-act loop on the DeskVNC MCP server

The `dvv` MCP server exposes one loop: open a saved machine, take the wheel,
look at the screen, act on it, and let the result flow back into the next
decision. Because the connection is the ordinary VNC, RDP or SSH one,
nothing is installed on the remote target and a person can take the wheel
back at any moment.

The loop is short enough to fit inside an agent's own think step:

```
observe  -> read screen and geometry, hold on to the generation token
decide   -> pick the next action the model would have picked anyway
act      -> settle one intent, never fire and forget
```

Three properties make this more than a hand-rolled screenshot-and-send:

* The observation is fenced twice. Geometry generation stops a click landing
  in the wrong place after a resize, content generation stops a keystroke
  landing in a window that appeared over the one the agent was looking at.
* The action is settled. Every intent gets an id and exactly one result, so
  an agent never waits forever on something a driver could not serve.
* The loop can lose the machine mid step. A person can take the wheel
  between observe and act; control is leased, and on any lease change the
  plane releases all held keys so a half-finished drag cannot strand the
  desktop.

## Working example, in JSONC

```jsonc
// 1. Perceive the surface: see the hosts available and open one
dvv_hosts   {}
dvv_open    {"hostId": "<id>", "perceive": true}

// 2. Acquire the wheel on the fresh limb
dvv_control {"limbId": "...", "action": "acquire"}

// 3. Look at the screen, capture the generation
dvv_screen  {"limbId": "...", "form": "full", "scale": 0.25}

// 4. Act against the same geometry
dvv_click   {"limbId": "...", "x": 700, "y": 400, "generation": 1}

// 5. Look again, then type into whatever now has focus
dvv_screen  {"limbId": "...", "form": "damage-crop"}
dvv_type    {"limbId": "...", "text": "notepad", "wpm": 3000}
dvv_key     {"limbId": "...", "keys": "meta+r"}
```

Every block above comes from the shipped build. The `perceive: true` flag
asks `dvv_open` to return the limb id, the screen size, and the session
state in the same call. `form: "full"` returns the whole frame, `form:
"damage-crop"` returns just the region that changed since the last read,
`scale: 0.25` downsamples to a faster, model-sized image.

## Generation fencing, the same on every coordinate

Every `dvv_screen` and `dvv_status` reports two numbers: a geometry
generation and a content generation. The first fences coordinates, the
second fences text. Carrying them back into the next call is the whole
pattern.

```
dvv_screen  {"limbId": "L", "form": "full", "scale": 0.25}
  -> { "geometry_generation": 7, "content_generation": 3, ... }

dvv_click   {"limbId": "L", "x": 700, "y": 400, "generation": 7}
  -> ok                          // geometry generation matches

dvv_click   {"limbId": "L", "x": 700, "y": 400, "generation": 6}
  -> STALE_GEOMETRY              // screen has since resized
```

A click computed against a stale screen is refused rather than landing
somewhere unintended. The same fence runs on text:

* `dvv_type` and `dvv_key` are refused with `SCREEN_CHANGED` when
  something window-sized has repainted since the agent last read pixels.
* The refusal is raised on a limb the agent has never read at all.
* One `dvv_screen` clears the fence. `dvv_status` reads no pixels, so it
  does not.
* There is no override.
* Terminal limbs are not fenced, because a PTY echoes what it is sent into
  a stream the agent reads back.

## Per-machine lease model

Every open machine is its own limb with its own lease, so an agent runs as
many loops as it has machines. N machines are N independent loops from
one agent. The README records a working example: one agent holding two
desktops and typing a different sum into a calculator on each finished
both in 0.95 seconds.

Two things follow from this:

* A limb id belongs to the connection that opened it. Hold one long-lived
  peer for the task rather than spawning a new `dvv` per call. A
  per-command process attaches and detaches each time, so the limb from
  the last call is gone by the next one.
* A person can revoke the lease from any agent-driven pane with one click
  and hand it back the same way. On any lease change, held keys and
  buttons are released. On `LEASE_REVOKED`, the right move is to observe
  again, not retry the previous action blindly.

Input travels over the protocol, so the local mouse and keyboard are never
involved. The viewer can be minimised while a person works on their own
computer, and the agent keeps driving on the remote.

## Latencies, measured on a real machine

Measured against a 1920x1080 Windows desktop over a LAN, on the shipped
build:

| call | result |
| --- | --- |
| `dvv_open` and attach | 4 ms |
| `dvv_control` acquire | under 1 ms |
| `dvv_screen` at `scale: 0.25` (112 KB) | 25 ms |
| `dvv_screen` at full scale (1.5 MB) | 70 ms |
| one observe-then-act cycle | 19 ms, about 52 actions a second |
| `dvv_type` throughput at `wpm: 12000` | 447 characters a second |

Two practical notes from the same workflow:

* Use `scale: 0.25` unless the task actually needs to read small text.
  That is what makes the observe-then-act cycle fit in 19 ms.
* Raise `wpm` when the task wants throughput rather than human-looking
  typing. 447 characters a second is the measured ceiling.

## Four rules the plane enforces

1. Attach before you act. Hold one long-lived peer for the task.
2. Coordinates are fenced with the geometry generation from the most
   recent read. Stale clicks are refused.
3. Typing is fenced too. `dvv_type` and `dvv_key` are refused with
   `SCREEN_CHANGED` when something large has repainted since the agent
   last looked. There is no override.
4. A person can take the wheel at any moment. Click into the pane and the
   agent is fenced out; held keys and buttons are released.

## Links

* Project: <https://github.com/psmux/DeskVNC>
* Integration notes: <https://github.com/psmux/DeskVNC/blob/main/docs/AGENTS.md>
* Page: <https://psmux.github.io/deskvnc/>
