# Protocol cores in DeskVNC

The three protocols that the project speaks natively (VNC, RDP and SSH) are
implemented as separate Rust crates rather than wrapped around an existing C or
C++ library. That choice shapes everything else: it keeps the parser code free
of `unsafe`, gives the MCP server (`dvv`) a clean surface to call into, and
allows each protocol to evolve against the servers it actually meets in the
field. The full protocol feature list, the security model, and the crate
layout live in [`docs/ARCHITECTURE.md`](https://github.com/psmux/DeskVNC/blob/main/docs/ARCHITECTURE.md).
This article goes deeper on the engineering substance behind the cores
themselves.

## The VNC core: RFB written from scratch

The RFB implementation in `vnc-core` covers protocol versions 3.3 through 3.8
with version negotiation. All the common encodings are present, with their full
option surface, not a subset:

- **Pixel encodings**: Raw, CopyRect, RRE, Hextile, Zlib, ZRLE, Tight, and Open
  H.264. ZRLE and Tight carry their own negotiated zlib streams; the binary
  reader treats each with its own framing rather than reusing the Zlib path.
- **Pseudo-encodings**: Cursor, Cursor With Alpha, X Cursor, VMware Cursor,
  Desktop Size, Extended Desktop Size, Desktop Name, Extended Clipboard,
  Fence, Continuous Updates, LastRect, Extended Mouse Buttons, plus the QEMU
  extensions for scancode keyboard input, keyboard LED state, and pointer
  motion events. Continuous Updates in particular avoids the classic VNC
  request-rect race by tracking server-side damage regions.
- **Adaptive quality**: Auto, High, Medium, Low, and Black and White presets.
  The Auto preset measures throughput on each frame interval and adjusts JPEG
  quality and subsampling, so a busy window drops quality gracefully instead
  of stalling.

H.264 deserves its own paragraph. The framing and decoder-context bookkeeping
live in `vnc-core`, but the video decode itself runs in the webview through
WebCodecs, where it picks up hardware acceleration through VideoToolbox on
macOS, D3D11 on Windows, or VAAPI on Linux. That split keeps the protocol
crate free of platform decode libraries and lets every supported host benefit
from the GPU path the webview already exposes.

### Authentication and transport

The security negotiation path covers **None**, **VncAuth**, **VeNCrypt**
including its X.509 subtypes, **RealVNC RSA-AES (RA2)**, **Apple
Diffie-Hellman** as used by macOS Screen Sharing, **MS-Logon**, and Tight
security negotiation. For each, the parser is the place where the protocol
shows its age, and VncAuth in particular truncates passwords to eight bytes
because the original RFB DES challenge response does. VeNCrypt, RA2 and the
Apple DH exchange are preferred where the server offers them.

TLS comes from `rustls`, with trust-on-first-use certificate pinning. A
changed certificate is surfaced rather than silently accepted; the pinned
trust store is the same one used for any SSH host key the same machine is
configured for, so a connection that has been trusted once stays trusted.
The certificate change is otherwise a quietly dangerous failure mode for a
client the user is likely to leave unattended.

The VNC connection can also ride inside an SSH `direct-tcpip` channel, which
does two useful things at once: it reaches VNC servers bound to the remote
machine's own loopback, and it encrypts the session on servers that offer no
TLS. The channel is the session's byte stream, so no local forwarded port is
ever opened; nothing else on your machine can reach the remote desktop. SSH
authentication uses a saved passphrase, your `ssh-agent`, or a key file, and
the host key is pinned on first contact.

### Filed off against real servers

The VNC core has been exercised against five very different server
implementations, and each one surfaced something worth fixing. **x11vnc**,
the most forgiving, still throws edge cases around X server extensions and
non-standard pseudo-encodings. **TigerVNC** is the source of truth for Tight
and ZRLE and exposes subtle diffs in how `LastRect` frames must be counted.
**QEMU's** built-in VNC is unusual because it relies on the QEMU key, LED
state, and pointer motion pseudo-encodings, which is also why they are in
the core at all: dropping them would silently break QEMU guests.
**RealVNC** exercises the RA2 authentication path that no open source server
implements. **macOS Screen Sharing** exercises the Apple Diffie-Hellman
exchange that no other server offers. Each one is its own regression target,
and the test matrix grows with every interoperability report that lands in
the issue tracker.

## The RDP core

The RDP core covers the Windows desktop case. Network Level Authentication
(NLA) is supported, so a connection is authenticated before a session is
exposed to the server, which matters on Windows machines that refuse pre-auth
logons. RemoteApp is supported, so an agent or a user can launch a single
application rather than the whole desktop, which keeps the surface area
small. Resolution control follows both directions, with the server resizing
through the standard dynamic virtual channel handling rather than tearing
the session down and starting again. Codec coverage spans the usual set:
RDP 8 compression, Progressive, and the variants the project has needed
against modern Windows hosts. The same `remote-core` session contract used
by the VNC core drives the RDP side, so the frontend, the MCP server, and
the lease model see the same shape regardless of protocol.

## The SSH core

SSH is its own crate (`vnc-files`, which also handles the SSH-tunnel path
for VNC) using `russh`. The terminal implementation survives a network
drop with a transparent reconnect, so a shell that lost its connection
mid-command comes back when the network does, with the user's history
preserved. SFTP transfers share the same host profile and the same pinned
trust store as the terminal session, with a dual-pane browser and
drag-and-drop upload that validates remote filenames before using them to
build local paths. Tunnels flow through the same `direct-tcpip` channel as
the VNC-over-SSH path, so a saved host can be a starting point for a
jump host chain. PuTTY key files are accepted as well, which keeps the
onboarding path short for anyone migrating from another SSH client.

## The shape that ties the cores together

Two invariants keep the design consistent. `vnc-core` does not depend on
Tauri, which means the parser and state machine are testable without a
system webview and that the frontend can be replaced without touching
protocol code. Whole framebuffers never cross IPC: decoded pixels travel
as binary dirty-rect messages over a Tauri channel and land directly in a
single WebGL2 texture, which is what makes the renderer bottleneck move
out of the way of the MCP control plane.

The second invariant is the `#![forbid(unsafe_code)]` attribute on every
crate that parses bytes from a remote peer: `remote-core`, `remote-pixel`,
`vnc-core`, `vnc-transport`, `vnc-discovery`, `vnc-store`, and
`vnc-files`. The only `unsafe` in the workspace lives in
`vnc-input-capture`, which wraps platform input APIs and never sees bytes
from the wire. Treating that boundary as the trust boundary, with a single
audited module on one side and a proof-free protocol parser on the other,
is what makes the rest of the workspace boringly safe to read.

For the full feature list, the security model, and the crate layout, see
[`docs/ARCHITECTURE.md`](https://github.com/psmux/DeskVNC/blob/main/docs/ARCHITECTURE.md).
