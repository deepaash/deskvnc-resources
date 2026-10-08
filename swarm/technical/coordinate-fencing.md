# Coordinate fencing in the DeskVNC agent plane

The agent plane in `DeskVNC` is built around a coordination contract that
makes every read and every act a transactional pair. The MCP server
(`dvv`) treats the moment of action as the moment that has to be backed
by proof: proof that the screen the agent is acting on is the screen
the agent last looked at, proof that the keystroke the agent sends is
the keystroke the agent meant to send, and proof that a person at the
keyboard can step in at any time. The result is a plane that an
operator can leave to an agent for hours, with the assurance that the
agent cannot drive against a stale screen.

## The generation field: why a click carries a token

Every screen the agent reads through `dvv_screen` carries a `generation`
counter, and every click the agent sends through `dvv_click` carries the
counter back as a `generation` argument. The counter advances each time
the remote desktop changes by enough to matter. When a click arrives, the
MCP server checks whether the counter it carries still matches the
machine's current generation. If it does not, the click is refused rather
than being executed against a screen the agent has not seen.

That is the whole idea, and the consequence is the point. A click
computed against a stale screen goes nowhere, on the grounds that
clicking is an irreversible action: the wrong window with the wrong
context can close a session, throw away an unsaved draft, or send a chat
message from the wrong account. The agent is forced to look again, redo
its targeting, and issue a click that is honest about what it knows.
This is what "read the screen, then act" means in code: the read carries
a token, the act carries the same token, and a server checks the match.

## The imageSpace line and coordinate conversion

A `dvv screen` invocation prints an `imageSpace` line that tells the
agent what coordinate system it is being shown. The default and the
recommended shape is to scale the captured image down so the picture the
agent reasons over is smaller than the remote pixels. With
`--scale 0.5`, for example, a point at `(mx, my)` on the picture is at
`(mx*2, my*2)` on the remote machine. With `scale: 0.25`, the captured
image is one quarter of the linear resolution, and the same conversion
applies at the new ratio. The agent converts before it clicks.

That last step is the one that goes wrong most often, which is why the
default scaling is `0.5` rather than `1.0` and why the conversion is
documented in the skill the agent consumes. A captured image at
`0.25` is roughly 100 KB for a 1920x1080 screen, which fits comfortably
in an LLM's context budget and reduces the cost of asking the agent to
re-read the screen every time something moves.

## Typing and keys: fenced until the screen is fresh

A click is only one of the inputs that can land somewhere wrong. Typing
and key presses do too, because a remote desktop changes focus when a
window opens, and a keystroke that was right five seconds ago may now be
typing into a different field or a different app. For typing and keys,
the plane applies the same rule with a tighter trigger. `dvv_type` and
`dvv_key` are refused if the screen has changed significantly since the
agent last looked, and the refusal returns the error code
`SCREEN_CHANGED`. The agent reads the screen, recomposes the keystroke
or the typed text against whatever is in focus now, and retries.

There is no override for this refusal. The plane is built on the idea
that the agent should not be able to force input past a screen it has
not seen, because the failure modes of doing so are exactly the failure
modes that take automation off the rails. The harder path, look again
and retry, is the correct one.

## Two errors worth knowing by name

The plane returns a small set of error codes that an agent ought to know
how to recover from. They are documented in the
[`dvv` skill](https://github.com/psmux/DeskVNC/blob/main/skills/deskvnc/SKILL.md),
and the recovery paths matter more than the names.

`SCREEN_CHANGED`. The screen has changed enough that a click, a key,
or a typed string would be unsafe against the agent's current model.
The agent's response is to read the screen again, then retry the
operation. A click that is refused this way never reached the remote
machine, so the agent can re-target it freely. A keystroke that is
refused this way never pressed anything, so the agent should re-read the
screen and decide whether the same shortcut still makes sense: if a
window opened on the remote side, the keyboard target may be the new
window and not the old one.

`LIMB_GONE`. The session the agent was driving has ended, usually
because the local user closed the window or because the session
otherwise went away. The agent's response is to call `dvv_limbs`, see
what is currently attached, and reopen the machine. A limb id belongs
to the connection that opened it, so `LIMB_GONE` can also mean the
agent is using a new process per call. The recommended pattern is to
use one long lived peer for a task.

## Why this is a feature, not a guard rail

The fencing is what makes the agent plane trustworthy. A human or a
script that uses the same VNC or RDP connection from the keyboard is
acting on coordinates the user is already looking at; an agent has to
earn that trust by handing a proof back with every action. The plane
ties the read and the act together as one transactional pair: the
agent reads, gets a token, and only then acts by handing the same token
back. When the token does not match, the act does not run.

This is also what makes the architecture scale beyond one agent acting
alone. With a lease per machine and a generation per screen, two
operators can hand a session back and forth without one of them driving
against a stale view. With a person able to click into the pane and take
control back at any moment, the system accommodates a human interruption
without dropping the audit trail. Both halves of that story rely on the
same idea: the agent has to be honest about what it knows, and the
plane has to make it cheap to be honest.

For the full life of a control plane call (the four-call loop, the
lease model, the shell fallback) see the
[`dvv` skill](https://github.com/psmux/DeskVNC/blob/main/skills/deskvnc/SKILL.md)
and the [project README](https://github.com/psmux/DeskVNC).
