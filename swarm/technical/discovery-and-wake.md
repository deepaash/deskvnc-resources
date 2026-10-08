# Discovery and Wake-on-LAN in DeskVNC

A remote desktop client that only knows about machines whose addresses
the operator typed in by hand is half a product. The other half is
finding the machine again on a network that may have renumbered its
addresses overnight. `DeskVNC` ships discovery as a first-class
subsystem in `vnc-discovery`, with three independent mechanisms (mDNS,
subnet scanning, and protocol-agnostic name resolution) and a
Wake-on-LAN path that fits into the same flow. The architecture and
behaviour are recorded in detail in
[`docs/ARCHITECTURE.md`](https://github.com/psmux/DeskVNC/blob/main/docs/ARCHITECTURE.md)
and the [project README](https://github.com/psmux/DeskVNC).

## mDNS browsing, the local segment

mDNS browsing listens for service records on the local multicast group,
filtering on two service types: `_rfb._tcp` for VNC servers that publish
themselves, and `_ard._tcp` for macOS Screen Sharing (Apple Remote
Desktop) hosts that publish their own records. The browse returns the
host name, the port, and any TXT-record text the publisher chose to
expose, which is enough to populate the host library without a single
manual entry on a network where servers are configured to advertise.

The local segment is the only place mDNS travels, by design. That gives
two useful properties: the answerer is almost always a machine on the
same broadcast domain, and the cost of being wrong (caching a stale
name) is bounded by how fast the network re-renumbers. The browse loop
is short and reissues its query on a sensible cadence, because silent
service churn on a long-lived client is the usual source of stale state.

## Subnet scanning with banner fingerprinting

The operator can also kick off a subnet scan with the **Scan network**
button in the host library or via the command line. The scan walks the
configured interface's IP range (with a configurable cap and an
opt-out for ranges the operator knows are not theirs) and, where it
finds something listening on the VNC or RDP ports, reads the protocol
banner.

The banner is what gives the scan specificity. A bare TCP connect tells
you nothing about what was on the other side; reading the RFB version
handshake tells you which VNC dialect it speaks (3.3 through 3.8, with
the negotiation in between) and lets the scan label the host with the
correct server family. **TigerVNC**, **x11vnc**, **RealVNC**, **QEMU**
guest consoles, and the **macOS Screen Sharing** server each have a
recognisable banner, and the fingerprint matching steers the host
library entry toward the right connection profile without manual
intervention.

The scan is rate-limited. Shared networks do not appreciate a host
walk that hits every address at line speed, and the cap and the pacing
are tuned so that scanning a /24 is not visible to the network as a
storm. The same scan will pause on a configured cap so the operator
can choose a different range when the one they picked is someone
else's.

## Name resolution over four protocols

A host's address is not always the right way to find it. A machine that
the operator knows as `frontdesk` may have moved from one network to
another, or the operator may not know the address at all and only know
the name. `DeskVNC` therefore resolves names through four protocols
rather than one:

- **mDNS** for `.local` names on the local segment, the same browse that
  surfaces services.
- **LLMNR** for Microsoft Windows networks that broadcast name queries
  on the local segment when DNS is unavailable.
- **NetBIOS** for legacy Windows name resolution, particularly useful
  on networks where older workstations and headless servers still
  register themselves.
- **MS-RPC** for Active Directory environments where the canonical name
  for a host is best found by asking the domain controller.

The benefit is that the host library shows real names where the network
already knows them, even when those names are not in DNS. The cost is
that every one of these answers can be forged by a host on the network,
which is why every response is parsed as hostile input. The names that
appear in the interface are the result of whatever the network said, and
the scanner and resolver exercise the same care a robust DNS resolver
does: length-checked, character-set-checked, and never trusted to mean
anything more than what the resolver asked for.

## Wake-on-LAN, including during reconnect

A machine that the operator looks after is often one that is asleep.
`DeskVNC` ships Wake-on-LAN as a first-class action: pick a host, hit
wake, and the magic packet goes out to the MAC address recorded with
the host profile. The packet is the standard AMD format with the
synchronisation stream and sixteen copies of the target MAC, sent as a
UDP broadcast on the configured interface.

What makes the design worth describing is what happens at reconnect
time. When an attempt to attach to a saved machine fails because the
remote side is not responding, the client offers to send a magic packet
and retry on the standard backoff schedule. The same code path is used
when a connection that was already up drops and the operator asks for
reconnect. In both cases the action stays inside the same flow the
operator is already in, rather than asking them to drop into a separate
tool to wake the machine up first.

## How these layers compose

The four mechanisms are independent. mDNS finds hosts that publish
themselves on the local segment. The scan finds hosts that listen but
do not advertise, using banner fingerprinting to label them well. The
name-resolution layer lets the operator find a machine by name on four
different kinds of network. Wake-on-LAN is the action that takes a
known machine and tries to bring it online without a separate tool.
Each one is its own subsystem; together they cover the practical shapes
a mixed estate (a Mac running Screen Sharing next to a Windows host
joined to Active Directory next to a Linux box running TigerVNC on a
flat network) takes.

For the implementation summary, the crate layout, and the parser
invariants that hold across these modules, see
[`docs/ARCHITECTURE.md`](https://github.com/psmux/DeskVNC/blob/main/docs/ARCHITECTURE.md).
