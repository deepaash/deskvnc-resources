# Why "at the protocol level" is the only thing that matters for an agent

Every tool that lets an AI agent drive a remote desktop has to answer one
question. Where does the agent's input go?

The funded agent products answer it by installing an agent on the remote
machine, and that answer is the reason they cannot reach the machines you
most want an agent on. Citrix, VDI, jump hosts and client owned desktops
all refuse installs, and that is the entire class of machines the agent
should be most useful on.

DeskVNC answers the same question differently. The agent's input goes
through the connection you already have, the same VNC, RDP or SSH
connection a person uses, with nothing installed on the remote machine.
The rest of this page is the mechanism, the consequences, and the
practical limits.

## The mechanism, in one paragraph

DeskVNC's `dvv` MCP server is a thin control plane over the three
protocol cores that are already in the client. The VNC core speaks RFB
3.3 to 3.8 with VeNCrypt, RA2 and Apple authentication, and has been
interoperated against x11vnc, TigerVNC, QEMU, RealVNC and macOS Screen
Sharing. The RDP core covers Windows desktops with NLA, RemoteApp,
resolution control and the usual codecs. The SSH core gives a terminal,
SFTP, tunnels and PuTTY key files. None of that is new, and none of it
requires a counterpart on the remote side beyond what the remote
operating system already provides. The new part is that an agent can ask
the client to open one of your saved machines, take the wheel on that
connection, look at the screen, click, type, and read the screen again,
all through those protocol cores, with the local mouse and keyboard
never involved.

## Why that is the structural difference

A typical agent tool installs a peer on the remote machine. The peer
joins a control plane, accepts commands over a long lived outbound
channel, and renders those commands into the desktop. The install itself
is the part that does not work on the machines the agent should be most
useful on. The list is well known: Citrix published apps refuse
unsanctioned installs. VDI images are sealed and rebuilt on a schedule.
Jump hosts are deliberately minimal. Client owned desktops are not yours
to install on. A protocol the machine already speaks gets in anyway,
because nothing is being installed, the remote is just doing what it
already does.

That is the structural reason the protocol level approach is the right
shape for an agent. It is not a feature flag and it is not a workaround
on top of an installed agent. The protocol is the path.

## The four rules the plane enforces

The protocol level approach gives the agent real reach, and it also
makes the control plane responsible for things a person handles
instinctively. The README documents four rules the plane enforces, and
they are worth reading once rather than debugging later.

**Attach before you act.** A limb id (the per machine handle the agent
gets back from `dvv_open`) belongs to the connection that opened it, so
use one long lived peer for a task rather than a new process per call.
This is the agent side equivalent of sitting down at the desk.

**Coordinates are fenced.** Screenshots carry a geometry generation, and
clicks carry the generation they were computed against. A click computed
against a stale screen is refused instead of landing somewhere unintended.
This is what makes "look, then act" safe across a window resize or a
session reconnect.

**Typing is fenced too.** `dvv_type` and `dvv_key` are refused with
`SCREEN_CHANGED` if something large has repainted since the agent last
looked, because focus moves when a window opens. There is no override.
The agent has to look again before it types.

**A person can take the wheel at any moment.** Click into the pane and
the agent is fenced out of that session. Held keys and buttons are
released, so a half finished drag cannot strand the desktop. Every
machine is its own limb with its own lease, so this is per machine, not
per session.

These four rules are how the plane stays safe to leave running.

## The performance ceiling

The protocol cores are written in Rust, the picture is decoded in Rust
and painted through WebGL2, and whole frames never cross the process
boundary. H.264 decodes with hardware acceleration where the webview
offers it. The numbers in the README are measured against a real
1920x1080 Windows desktop on a LAN, and they are the reason a loop fits
inside an agent turn:

| call | time |
| --- | --- |
| `dvv_open` and attach | 4 ms |
| `dvv_control` acquire | under 1 ms |
| `dvv_screen` at `scale: 0.25` (112 KB) | 25 ms |
| `dvv_screen` at full scale (1.5 MB) | 70 ms |
| one observe then act cycle | 19 ms, about 52 actions per second |
| `dvv_type` throughput at `wpm: 12000` | 447 characters per second |

Every machine is its own limb with its own lease, so ten machines are ten
independent loops. The README reports one agent holding two desktops and
typing a different sum into a calculator on each, finishing both in
0.95 seconds. The local mouse and keyboard are never involved, the
window can be minimised, and the operator carries on using their own
computer.

## What this lets you do

The shape that opens up is the one the funded agent tools cannot reach.

- Drive Citrix and VDI desktops where installed software is refused.
  VNC and RDP are the only paths in, and DeskVNC speaks both.
- Drive jump hosts and bastions through a single ordinary SSH session,
  including the terminal and `dvv_run` for command execution.
- Drive client owned machines you cannot install on, by connecting to
  the remote through a Boundary support invitation or through a VNC or
  RDP server the client already runs.
- Drive a homelab of mixed operating systems through one client and one
  control plane, with the host library and live thumbnails as the
  default agent shortlist.
- Hand control back to a person mid task, the moment a human decides to
  look. The agent waits, the held keys release, and a person finishes
  the case.

The pages under use cases walk through each of these in real syntax.

## What this is not

The protocol level approach is the right shape for an agent on machines
that refuse installs. It is not a replacement for the operational
tooling a managed estate already runs. A MeshCentral server with agents
on every device is still the right tool for fleet inventory, scheduled
scripting and continuous monitoring. A Guacamole gateway is still the
right tool for browser based delivery of remote desktops to a large
user base. DeskVNC's `dvv` is one piece of the picture, the piece that
fits where the other pieces cannot go.

## Where to go from here

- [`comparison-tools.md`](comparison-tools.md): the table view, with
  every cell checked against each project's own materials.
- [`usecase-vdi-citrix.md`](usecase-vdi-citrix.md): the Citrix and VDI
  case in real MCP syntax.
- [`usecase-jump-hosts.md`](usecase-jump-hosts.md): jump hosts and
  bastions, including `dvv_term_read`, `dvv_term_send` and `dvv_run`.
- [`usecase-homelab.md`](usecase-homelab.md): a mixed homelab of
  Windows, macOS and Linux, with Wake on LAN and network discovery.
- [`usecase-human-takeover.md`](usecase-human-takeover.md): the per
  machine lease, takeover mid task, and the approval story.
