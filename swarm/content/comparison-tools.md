# Remote desktop tools compared

A side by side of the tools people evaluating DeskVNC usually run into, with
the dimensions that actually drive the choice. Every cell below is a claim
about a real, public product, and sources for each tool are listed at the
bottom of the table.

Capabilities change. Versions, editions, and licensing for every product in
this table shift release over release, and the table reflects what was
verifiable at the time of writing. Treat this page as a starting point, then
verify the current state of each product on its own project page before
making a decision. DeskVNC itself is under active development and the same
applies.

## The short version

DeskVNC is one native client for VNC, RDP and SSH on Windows, macOS and
Linux, with credentials in the operating system keychain, a saved host
library with live thumbnails, network discovery and Wake on LAN, and an
optional control plane that lets an AI agent drive the same connections a
person would, with nothing installed on the remote target and a person able
to take the wheel back at any moment. The other tools in this table each do
part of that, and each does the parts it does well.

## The table

| Dimension | DeskVNC | RustDesk | MeshCentral | Apache Guacamole | RealVNC Connect | TigerVNC | Windows built in RDP client |
| --- | --- | --- | --- | --- | --- | --- | --- |
| What it is | Native desktop client for VNC, RDP and SSH, with an optional AI agent control plane | Open source remote desktop application, self host or use the public rendezvous server | Open source web based remote monitoring and management server with agent installed on each managed device | Clientless remote desktop gateway, served to a browser, translates VNC, RDP and SSH to a Guacamole protocol stream | VNC Server plus VNC Viewer, using the RFB protocol, with optional cloud relay | Open source VNC client and server, fork of TightVNC | Built in client `mstsc.exe` for the Microsoft Remote Desktop Protocol |
| Connects to | VNC (RFB 3.3 to 3.8, VeNCrypt, RA2, Apple auth), RDP, SSH | Other machines running the RustDesk client (its own protocol over a self hosted or public relay) | Devices that have the MeshAgent installed and registered to your server | Any VNC, RDP or SSH server the gateway can reach on the network | Any RFB compatible VNC server, including RealVNC Server, macOS Screen Sharing, TigerVNC and others | Any RFB compatible VNC server | Windows machines running the Remote Desktop Services server, plus third party servers such as xrdp and FreeRDP |
| What you install on the remote side | Nothing. DeskVNC speaks the protocol the remote machine already speaks. | A RustDesk client runs on the remote machine, peer to peer with the operator. | A MeshAgent on every managed device, registered to your MeshCentral server. | Nothing on the remote. The gateway runs `guacd` on a server you operate. | RealVNC Server on the remote machine (or the built in macOS Screen Sharing server, which speaks the same RFB). | A VNC server (such as `Xvnc`, `x0vncserver`, `w0vncserver`, or the unmaintained `winvnc`). | A Remote Desktop server built in to Windows Pro and Server editions, or third party servers like xrdp. |
| Platforms the client runs on | Windows, macOS, Linux | Windows, macOS, Linux, iOS, Android, Web client | Browser based, plus MeshAgent on Windows, Linux, macOS, FreeBSD, and Android | Any modern web browser, no plugin | Windows, macOS, Linux, Raspberry Pi, iOS, Android | Windows, Linux, macOS for the viewer; Linux for the server | Windows (the `mstsc.exe` binary); Microsoft also publishes Remote Desktop clients for macOS, iOS and Android |
| Deployment model | One client on your machine, connects out to your machines. No account, no server you run. | Self hosted rendezvous and relay server, or use the public one, plus clients on both ends. | You operate the MeshCentral server, agents on managed devices, browsers for operators. | You operate the gateway, including a Java web app, `guacd`, and a database. | A pair of apps on the remote and operator, with optional cloud relay through RealVNC's service. | Local clients and servers on each machine. | Built in to Windows, no server to run for the client side. |
| Licensing | Dual licensed MIT OR Apache 2.0 | GNU Affero General Public License v3 (AGPL 3), with a separately sold Pro server | Apache License 2.0 | Apache License 2.0 | Proprietary. Free Home tier for non commercial use, paid Professional and Enterprise tiers. | GNU General Public License v2 or later | Bundled with Windows. The protocol is proprietary, and Microsoft requires third party implementations to license the relevant RDP patents. |
| Maintenance | One engineer, plus external contributors, releasing regularly on a public changelog. Releases on GitHub. | Community maintained open source project, with a Pro server sold separately. | Community maintained, originally sponsored by Intel until 2022. | Apache Software Foundation top level project, with a long release history. | A commercial company, RealVNC Ltd, founded by members of the original AT&T VNC team. | Community maintained, primarily by Cendio AB for use in their ThinLinc product. | Maintained by Microsoft as part of Windows. |
| AI agent can drive it | Yes, the included `dvv` MCP server, over the same VNC, RDP or SSH connection a person uses, with nothing installed on the target. | No first party agent integration. RustDesk connects to other machines running the RustDesk client, so an automation would still need the remote side installed. | No first party agent integration. MeshAgent is the agent on the managed device, and the management surface is the browser console, not a protocol. | No first party agent integration. Guacamole is a browser gateway that translates VNC, RDP and SSH to its own protocol, and a custom integration could speak that protocol, but no shipped agent plane exists. | No first party agent integration. VNC Viewer is a graphical client that drives a desktop through RFB. | No first party agent integration. The TigerVNC viewer is a graphical client. | No first party agent integration. `mstsc.exe` is a graphical client. |
| Open source | Yes, MIT OR Apache 2.0 | Yes, AGPL 3 | Yes, Apache 2.0 | Yes, Apache 2.0 | No. The codebase contains a separate "VNC Open" component that is GPL, but the VNC Connect product is proprietary. | Yes, GPL 2 or later | No |

## How to read this

A few notes on the choices behind the rows.

**What you install on the remote side** is the single most important row
when an AI agent is in the picture. The funded agent products for Windows
desktops all install an agent on the desktop, which is the row that locks
the tool out of Citrix, VDI, jump hosts and client owned machines. Apache
Guacamole avoids that, because `guacd` reaches the remote over VNC, RDP or
SSH from a server you operate. DeskVNC avoids it the same way, but from a
native client on your own machine rather than a server in a data centre.

**Protocol coverage** matters because it sets the menu of machines a tool
can reach. RFB covers the wide world of VNC servers, RDP covers Windows
desktops and a few cross platform servers, and SSH covers Linux, network
gear and the inside of a jump host. DeskVNC is the only client in the table
that handles all three from one native window.

**Deployment model** matters because it sets who runs what. DeskVNC is a
single client that connects out, so there is no server you stand up to
operate the tool itself. MeshCentral and Guacamole are server products:
you run a server, and that is its own operational surface. RealVNC and
RustDesk both come in a self hosted flavour and a hosted flavour.

**AI agent can drive it** is the row that puts DeskVNC in a different
place. The `dvv` MCP server is included, uses the same VNC, RDP or SSH
connection a person would, and the remote machine has no agent and no
extra software. None of the other tools in the table ship a first party
agent control plane, and several of them require installed software at the
remote end, which is the structural reason the other tools cannot reach
machines that refuse installs.

## What each tool is good at

Every tool in this table has a real job it does well.

**RustDesk** is a self hostable remote desktop for direct operator to
remote-user support, with P2P connections and end to end encryption. It is
a strong fit when both ends of the call are on machines you can install
software on, and you want the rendezvous and relay under your own control.

**MeshCentral** is a full remote monitoring and management web site, with
inventory, scripting, terminal and file transfer on top of remote desktop.
It is a strong fit when the model is "every device has our agent, we
manage it through our web console", and when the operational surface of
running a server is part of the value.

**Apache Guacamole** is the reference implementation of a clientless remote
desktop gateway. It is a strong fit when the delivery surface is a browser,
when you want one central gateway that brokers VNC, RDP and SSH for many
users, and when the standard authentication extensions (LDAP, SSO, MFA)
matter to the deployment.

**RealVNC Connect** is the direct commercial descendant of the original
AT&T VNC work, with a cloud relay and signed installers. It is a strong
fit when you want VNC, you want a vendor behind the protocol, and you want
the platform integrations that come with a long lived commercial product.

**TigerVNC** is the open source VNC reference client and server, with
strong performance, encryption on every platform, and a thin dependency
footprint. It is a strong fit when you want VNC only, you want GPL, and
the machines you care about are Linux servers.

**The built in Windows RDP client** is the most direct way to reach a
Windows desktop, ships with the operating system, and integrates with
Windows credential management. It is a strong fit for the Windows to
Windows case, especially inside a domain.

## How DeskVNC fits

DeskVNC is one native client for VNC, RDP and SSH on the three desktop
operating systems, with the credentials in the operating system keychain,
a saved host library with live thumbnails, network discovery and Wake on
LAN, and an optional MCP control plane that lets an AI agent drive the
same connections. The case for it is the row that no other tool can fill
in the same shape: an agent control plane that reaches the machine over
the protocol the machine already speaks, with nothing installed at the
remote end and a person able to take the wheel back at any moment.

For the case for it in detail, see
[`comparison-why-protocol.md`](comparison-why-protocol.md). For the shape
of it in real situations, see the use case pages: VDI and Citrix
([`usecase-vdi-citrix.md`](usecase-vdi-citrix.md)), jump hosts
([`usecase-jump-hosts.md`](usecase-jump-hosts.md)), homelabs
([`usecase-homelab.md`](usecase-homelab.md)) and human takeover mid task
([`usecase-human-takeover.md`](usecase-human-takeover.md)).

## Sources verified while writing this page

The facts in the table were checked against each project's own materials
on the dates listed. Page readers should re check the current state, since
projects move.

- DeskVNC: <https://github.com/psmux/DeskVNC> (project README and docs).
  Verified against the live README on 2026-10-09.
- RustDesk: <https://github.com/rustdesk/rustdesk> and
  <https://rustdesk.com/docs/en/>. License shown as AGPL 3 in the GitHub
  repository sidebar. Verified 2026-10-09.
- MeshCentral: <https://github.com/Ylianst/MeshCentral> and the
  MeshCentral design and architecture guide. Verified 2026-10-09.
- Apache Guacamole: <https://guacamole.apache.org/>, the Apache Guacamole
  Manual v1.6.0, and the Wikipedia article. Verified 2026-10-09.
- RealVNC: <https://www.realvnc.com/> (the live site, the pricing page
  and the Wikipedia article were used together; some RealVNC URLs return
  a 403 to non browser clients, so the Wikipedia article was used to
  cross check platform and tier claims). Verified 2026-10-09.
- TigerVNC: <https://github.com/TigerVNC/tigervnc> and the Wikipedia
  article. Verified 2026-10-09.
- Windows built in RDP client: Wikipedia "Remote Desktop Protocol"
  article, cross checked against Microsoft documentation. Verified
  2026-10-09.
