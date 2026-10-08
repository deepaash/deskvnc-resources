# Why protocol-level remote control reaches machines that refuse installed agents

If you have ever tried to put an automation agent on a Citrix desktop, on a
VDI machine, on a jump host, or on a PC that belongs to a client, you have
already met the wall. The funded tools that drive a Windows desktop from an
AI agent all install an agent of their own on that desktop, and those four
classes of machine refuse to install anything. DeskVNC takes a different
shape: it speaks a protocol those machines already speak, so it gets in
without asking the desktop to change.

## The argument, in the project's own words

> The funded tools that let an agent use a Windows desktop all install an
> agent on that desktop, which is refused on Citrix, on VDI, on jump hosts
> and on anything a client owns. A protocol those machines already speak
> gets in anyway.

That sentence is the case for the project. The rest of the design follows
from it.

## What "protocol-level" means in practice

DeskVNC connects to a remote machine exactly the way every remote desktop
client already does. For Windows and most VDI estates it is VNC/RFB or
RDP; for Linux and any host with an SSH daemon it is SSH. The remote
machine never sees new software, never gets a new service, never asks for
elevation, never raises the kind of change-control question that stops an
automation project before it starts. Nothing is installed on the target.

That matters for the exact four classes of machine above, because those
classes are where the automation work wants to go. A protocol those
machines already speak is a door that is already open.

## What an AI agent gets once it gets in

The same protocol connection that a person uses by hand is exposed through
`dvv`, an MCP server that ships inside DeskVNCViewer. The agent opens one
of your saved machines, observes the screen, and sends clicks and
keystrokes, all over the VNC, RDP or SSH connection. Because the
connection is the ordinary one:

* the agent runs at a measurable 4 ms attach, under 1 ms acquire, 25 ms
  per screenshot at `scale: 0.25`, and 19 ms per observe-then-act cycle
  on a 1920x1080 Windows desktop over a LAN;
* every open machine is its own limb with its own lease, so an agent runs
  as many loops as it has machines, and ten machines are ten independent
  loops;
* a person can take the wheel at any moment with one click on the pane,
  held keys and buttons are released, and the agent is fenced out of
  that session until it is handed back;
* the local mouse and keyboard are never involved, because input travels
  over the protocol, so a person carries on using their own computer
  while the agent works on a remote one.

## Where it fits

The shape suits anyone whose automation target list includes at least one
class of machine that refuses an installed agent:

* **Citrix and VDI estates** that are managed to a policy that says "no
  extra software on the desktop".
* **Jump hosts** that proxy into the rest of a network and are themselves
  a friction surface for change.
* **Client-owned desktops** at an MSP, a support shop, or a consultancy,
  where the customer's policy says nothing new touches their machine.
* **Legacy Windows estates** that have to stay on the manufacturer's own
  remote tool, and can still be driven through that tool's VNC or RDP
  endpoint.

## The shape, in one line

DeskVNC is the same connection a person uses by hand, with a control
plane that lets the agent ride along on it. The remote machine keeps
its policy, its locked-down image, and its change-control process. The
agent reaches it through a protocol the desktop already trusts.

## Links

* Project: <https://github.com/psmux/DeskVNC>
* Page: <https://psmux.github.io/deskvnc/>
