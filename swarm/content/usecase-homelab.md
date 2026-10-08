# Use case: a homelab of mixed operating systems

A homelab is the case where the operator owns the machines and the
network, and the operator is also the person who fixes the machines
when they go wrong. The shape of the lab changes over time. A Windows
desktop for the day job. A Mac for design and personal use. A Linux
server in the cupboard that runs the home services. A Raspberry Pi
under the desk. A NAS on the shelf. Each machine speaks a different
protocol by default, each machine has its own credentials, and each
machine has its own state when you last saw it. The host library in
DeskVNC is built for exactly this case, with live thumbnails of every
saved machine, network discovery for the machines that are listening
nearby, and Wake on LAN for the machines that are asleep.

This page is the homelab shape in real syntax, with the discovery, the
thumbnail workflow, the Wake on LAN, and the agent loop an operator
can hand to a tool without the operator having to install anything
new on the remote side.

## The situation

You have a homelab that runs a mix of Windows, macOS and Linux
machines. Some are on all the time. Some sleep when they are not in
use. Some move around the network. The lab is the place where you
break things on purpose, recover them, and try the next idea. You want
one window that shows you every machine at a glance, with a thumbnail
of what the screen looked like when you last left it, so you can pick
the next machine to drive without remembering which one was in which
state.

## The host library and live thumbnails

The host library is the first surface an operator sees. Every saved
machine is a tile, and each tile carries a live thumbnail of the
desktop or terminal the operator last saw. The tiles are the operator's
shortlist, and they are also the agent's shortlist, because the host
library is the menu of machines `dvv_hosts` returns to the agent.

A tile in the library shows the name, the protocol, the host, the
credentials are in the keychain, and the thumbnail. The credentials
never leave the operator's machine. The protocol is the connection.

For the operator, the workflow is:

1. Press **New Host**, or paste an address into the bar at the top and
   press **Connect**, which saves nothing. The address can be written
   the way the operator already thinks of it: `10.0.0.4`,
   `10.0.0.4:5901`, `rdp://frontdesk`, `ssh://ops@jump-01`.
2. Set the credentials, which go into the operating system keychain.
3. Double click the tile to open the connection.

The same library is the menu the agent works from. `dvv_hosts` returns
the saved machines, and the agent picks one by id.

## Network discovery and Wake on LAN

Two features earn their place on a homelab: knowing what is listening
nearby without remembering addresses, and waking machines that are
asleep.

**Network discovery** is the operator pressing **Scan network** in
DeskVNC. The README documents the implementation: mDNS browsing,
polite subnet scanning with banner fingerprinting, name resolution
over mDNS, LLMNR, NetBIOS and MS-RPC. The result is a list of
machines the operator did not have to know about in advance, with the
protocol they are speaking and a likely identity.

The agent gets the same shape through the host library. Once a machine
is saved, it has an id the agent can use, and the agent does not need
to do the scan itself. For the operator, the scan is a one shot
"what is on the network right now" action that is useful when the
operator is setting up the lab, not when the agent is running a task.

**Wake on LAN** is the operator right clicking a tile for a machine
that is asleep and choosing **Wake**. DeskVNC sends the magic packet
to the machine's MAC address, the machine wakes, the operator opens
the connection. The agent can use the same primitive through the host
library, because the connection to a sleeping machine is a failure
that the plane recovers from, and the recovery path is the operator
sending the wake packet.

The combination is what a homelab wants: a list of every machine the
operator looks after, a thumbnail of each one, a one click path to
open it, and a one click path to wake it if it is asleep.

## The agent loop on a homelab

The agent gets the saved host list, picks a machine by id, and uses
the same four call loop the rest of this set of pages covers. The
homelab case is the one where the agent's `dvv_hosts` call is most
useful, because the homelab is the menu, and the menu is the shortlist
the operator curated.

```jsonc
dvv_hosts   {}
// -> [
//      {"id": "router",       "protocol": "ssh",   "host": "10.0.0.1"},
//      {"id": "nas",          "protocol": "vnc",   "host": "10.0.0.10"},
//      {"id": "macbook",      "protocol": "vnc",   "host": "10.0.0.20"},
//      {"id": "workstation",  "protocol": "rdp",   "host": "10.0.0.30"},
//      {"id": "pi-cluster",   "protocol": "ssh",   "host": "10.0.0.40"}
//    ]

dvv_open    {"hostId": "router", "perceive": true}
// -> {"limbId": "limb-3f1", "size": [80, 24], "state": "attached"}

dvv_control {"limbId": "limb-3f1", "action": "acquire"}
// -> {"ok": true, "lease": "lease-77a"}
```

For a Linux server in the cupboard, the calls are the SSH terminal
calls. For a Windows machine, the calls are the desktop calls. For a
Mac, the calls are the VNC desktop calls. The agent does not have to
care which protocol is in play, because the host library already
recorded the protocol and `dvv_open` selects the right one.

A practical example: the operator wants the agent to check the status
of the NAS and the Pi cluster, in parallel, without the operator
opening a window. The block below shows the shape of each call; the
full argument reference lives in the project integration notes at
`docs/AGENTS.md` and the compiled skill that `dvv setup` writes next
to each agent.

```jsonc
dvv_open    {"hostId": "nas", "perceive": true}
// -> {"limbId": "limb-4a1", ...}
dvv_run     {"limbId": "limb-4a1", "command": "uptime", "timeout": 5000}
// -> {"ok": true, "stdout": " 09:14:11 up 23 days,  4:17,  1 user,  load average: 0.08, 0.05, 0.01\n", "exit": 0}

dvv_open    {"hostId": "pi-cluster", "perceive": true}
// -> {"limbId": "limb-5b2", ...}
dvv_run     {"limbId": "limb-5b2", "command": "kubectl get nodes", "timeout": 5000}
// -> {"ok": true, "stdout": "NAME      STATUS   ROLES                  AGE   VERSION\npi-01     Ready    control-plane,master   67d   v1.30.0\npi-02     Ready    <none>                 67d   v1.30.0\n", "exit": 0}
```

Each machine is its own limb with its own lease, so the two checks
are independent loops and the agent can drive them in parallel.

## The closing summary

A homelab is the case where one client for every machine, with
credentials in the keychain, a saved host library, network discovery
and Wake on LAN, is the right shape on its own. The agent control
plane layers on top of that without changing the shape. The agent
uses the same menu of saved machines, the same credentials, the same
protocols, and the same wake on LAN path. The combination is an
operator's home setup that an agent can drive, with the operator's
approval, over the protocols the lab already speaks.

For the broader case for the protocol level approach, see
[`comparison-why-protocol.md`](comparison-why-protocol.md). For the
Citrix and VDI case, see
[`usecase-vdi-citrix.md`](usecase-vdi-citrix.md). For the jump host
case, with the SSH terminal calls, see
[`usecase-jump-hosts.md`](usecase-jump-hosts.md). For the per machine
lease and the human takeover story, see
[`usecase-human-takeover.md`](usecase-human-takeover.md).
