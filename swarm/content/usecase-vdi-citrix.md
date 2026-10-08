# Use case: an AI agent on Citrix and VDI

The machines an agent should be most useful on are exactly the ones that
refuse installed agent software. A published Citrix app is sealed by
policy. A VDI image is rebuilt on a schedule. A client owned desktop is
not yours to install on. The funded agent products, the ones that ship
as commercial products for Windows desktops, all install an agent on the
target, so they are the wrong shape for this case from the start.

This page is the shape that does fit, and the MCP syntax for an agent
working a Citrix published app or a VDI desktop.

## The situation

You have a fleet of Citrix published apps and a pool of VDI desktops.
The user facing experience is good and you do not want to change it. You
also want an AI agent to be able to do work on the same desktops, on the
same apps, in the same sessions, for the cases where automation saves
real time. The way the user reaches the desktop today is a Citrix
Workspace launcher or a Remote Desktop client pointed at a broker.
Those brokers terminate the protocol at the edge, and the desktop the
agent needs to drive is on the inside of that edge.

## Why installed agent tooling struggles here

The installed agent pattern is: drop a peer on the remote, the peer
joins a control plane, the agent on the operator side issues commands
to the peer, the peer renders them into the desktop. The install step
is the one that fails on this estate. The Citrix image is sealed, the
VDI image is rebuilt, the client owned desktop is someone else's
property. Even when an install is technically possible, the change
control process around it is the long pole, and a fleet of images with
custom agents is its own long term maintenance burden.

A protocol the machine already speaks gets in anyway. That is the
structural reason the protocol level approach is the right shape for
this estate.

## The protocol level approach with DeskVNC

DeskVNC's `dvv` MCP server does not install anything on the target. It
speaks the same VNC or RDP the user already uses, and the agent drives
the same connection. The Citrix published app or VDI desktop is the
remote; the user's DeskVNC on their own machine is the operator; the
agent connects through the same VNC or RDP server the user does.

For the agent, the loop is the same four calls the README documents:

1. `dvv_hosts` to find the saved machine.
2. `dvv_open` to open it and get a limb id.
3. `dvv_control` to take the wheel.
4. `dvv_screen`, `dvv_click`, `dvv_type`, `dvv_key` to do the work.

The agent only sees the protocol the remote exposes, which is exactly
the same protocol the human sees.

## A real session, in real syntax

The agent is working a Citrix published desktop that has been saved as
`finance-pubapp` in the host library. The agent wants to open Notepad,
type a copy of a number from a screen, and save the file.

```jsonc
dvv_hosts   {}
// -> [{"id": "finance-pubapp", "name": "Finance PubApp", ...}]

dvv_open    {"hostId": "finance-pubapp", "perceive": true}
// -> {"limbId": "limb-7a3", "size": [1920, 1080], "state": "attached"}

dvv_control {"limbId": "limb-7a3", "action": "acquire"}
// -> {"ok": true, "lease": "lease-91c"}

dvv_screen  {"limbId": "limb-7a3", "form": "full", "scale": 0.25}
// -> {"generation": 1, "png": "<base64>", "width": 480, "height": 270}

dvv_click   {"limbId": "limb-7a3", "x": 700, "y": 400, "generation": 1}
// -> click on the Start button at remote pixel coordinates
```

The agent reads the screen, sees the Start menu, and types the program
name. The `generation` field is what the plane uses to refuse a click
computed against a stale screen. The agent has to look again before it
acts if the screen repainted.

```jsonc
dvv_type    {"limbId": "limb-7a3", "text": "notepad", "wpm": 3000}
// -> {"ok": true, "settled": true}
dvv_key     {"limbId": "limb-7a3", "keys": "Return"}
// -> {"ok": true, "settled": true}

dvv_screen  {"limbId": "limb-7a3", "form": "damage-crop"}
// -> {"generation": 2, ...}
// Notepad is open. The agent reads what it needs.
```

When the agent wants to drive a published app that is itself a published
desktop (rather than a published single app), the loop is the same, the
host is just a different saved machine. The point is that the agent is
not asking the Citrix infrastructure for anything new. It is asking the
same VNC or RDP server the user is asking, with the same credentials,
through the same broker.

## Why this fits

Three properties come out of the protocol level approach that matter on
this estate.

**Nothing changes on the desktop.** The Citrix image is unchanged. The
VDI image is unchanged. The agent is just another client of the existing
VNC or RDP server. Change control stays the way it is, the image
rebuild cycle stays the way it is, and the agent inherits the existing
authentication, broker configuration and high availability.

**The same agent reaches every machine the user can reach.** A agent
that already drives a Windows desktop with `dvv` will drive a Citrix
published app, a VDI desktop, and a client owned machine through the
same set of calls, because the surface to the agent is the protocol,
not the deployment underneath.

**A person can take over at any moment.** The takeover semantics on
each machine are per machine, not per session. A support engineer can
click into the pane mid task, finish a step, and hand the pane back.
This is the property that makes an agent on Citrix approvable, because
the human is never locked out.

## The closing summary

Citrix and VDI estates are exactly the case where the protocol level
approach earns its place. An agent that wants to work on a published
app or a virtual desktop already has the protocol open, so the agent
can use the same path, with the same credentials, with no install on
the target, and with a person able to take the wheel back. The
combination is the reason the agent loop fits on this estate, and the
combination is the reason an agent can be approved to run on it.

For the protocol level argument in full, see
[`comparison-why-protocol.md`](comparison-why-protocol.md). For the
jump host case, where the protocol is SSH and the agent is driving a
terminal, see [`usecase-jump-hosts.md`](usecase-jump-hosts.md). For the
homelab case, with Wake on LAN and network discovery in the mix, see
[`usecase-homelab.md`](usecase-homelab.md). For the per machine lease
and the human takeover story, see
[`usecase-human-takeover.md`](usecase-human-takeover.md).
