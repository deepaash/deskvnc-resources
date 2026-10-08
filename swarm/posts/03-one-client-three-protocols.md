---
title: "Three remote desktop protocols, one client, and no more drawer of tools"
description: "VNC, RDP and SSH over the same Rust client, with credentials in the OS keychain. How a working day stops looking like a tray of half-pinned apps."
date: 2026-10-09
tags: ["vnc", "rdp", "ssh", "rust", "tauri", "remote-desktop", "consolidation"]
---

The tray was the giveaway. Looking after a non-trivial fleet, the dock at the bottom of the screen on a Tuesday morning had a VNC client, an RDP client, an SSH client, an SFTP client that only opened in pairs, a credential manager with a different password for each, and a small folder of `.rdp` shortcuts that had been there since the last office move. Six icons, four file formats, three password stores. The work was "open a session on a machine I look after", and it had been reinvented three times.

The honest fix is consolidation. VNC, RDP and SSH are three protocols that answer the same question: drive a remote session. They share a shape. Each one renders a frame and accepts typed input. The differences live at the wire level, not in the workflow.

[DeskVNC](https://github.com/psmux/DeskVNC) is a single native client that handles all three, written in Rust and built on Tauri 2. It runs on Windows, macOS and Linux, holds the saved hosts you actually look after, and keeps their credentials in the operating system keychain so the password store problem goes away with everything else. The [project page](https://psmux.github.io/deskvnc/) has the binary, screenshots and release notes for the same client.

## What one client looks like in practice

A saved host is a row in a library: name, protocol, address, user, optional key file, optional display overrides. That row is the whole configuration, and it is the only thing you have to write once. Open it and you get a tab, with a live thumbnail on the dock, a real frame in the window, and clipboard that round-trips. Open a second tab on a different machine and the two sit side by side in a split pane.

The interesting numbers live in the protocol implementation, not the chrome. The VNC stack speaks RFB 3.3 through 3.8, with VeNCrypt, RA2 and Apple authentication handled in the same binary. The RDP stack uses the same machinery underneath, with a connection sequence, a glyph cache, and clipboard channels that interop with the Windows clients you already have. The SSH stack gives you a real terminal plus a bidirectional SFTP pane, which is most of what an SFTP-only client ever did. Reconnection after a drop is automatic on all three, so a sleep on the laptop does not become a half-day of retyping credentials.

This is what consolidation actually buys. One client, one library, one update cadence. The Linux server, the Windows box, the Mac workstation and the jump host are all rows in the same view, and you reach any of them with the same gesture.

## The keychain half of the consolidation story

Putting all three protocols into one binary is only half the work. The other half is where the passwords live. DeskVNC stores credentials in the operating system keychain, not in a settings file: Credential Manager on Windows, Keychain Access on macOS, the Secret Service on Linux. That is the same place your browser keeps the credentials your browser knows about, and it is the place every credential tool already knows how to back up.

The practical effect is that opening a saved host is one click. The credentials fill in from the OS, the protocol layer negotiates, and the window comes up. There is no second prompt for a password because the right one is already authorised for this process, and there is no `.rdp` shortcut with a plain-text password because there never was a `.rdp` shortcut.

The agent layer that rides on top of the same client uses the same keychain entries. The MCP server inside the binary sees the same saved hosts the human sees, opens them the same way, and never has to be told a separate password for the same machine.

## One loop, three protocols

The MCP loop is identical regardless of protocol. `dvv_open` takes a `protocol` field of `vnc`, `rdp` or `ssh`, returns a lease handle, and the agent drives the machine the same way from there:

```json
{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{
  "name":"dvv_open",
  "arguments":{"machine":"design-srv-01","protocol":"rdp","user":"ops"}
}}

{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{
  "name":"dvv_open",
  "arguments":{"machine":"build-box-03","protocol":"vnc","user":"pi"}
}}

{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{
  "name":"dvv_open",
  "arguments":{"machine":"edge-fw-09","protocol":"ssh","user":"netops"}
}}

{"jsonrpc":"2.0","id":4,"method":"tools/call","params":{
  "name":"dvv_screen",
  "arguments":{"lease":"L-edge","scale":0.25,"format":"png"}
}}
```

Each call gets its own lease, so ten machines are ten independent loops. The agent that opens the SSH jump host for log inspection is not contending for input with the one driving the RDP build box. The four-call shape from the MCP post holds on all three protocols, which is the part of the design that lets the agent treat the estate as one workload instead of three.

## What the Tuesday morning looks like afterwards

Six icons collapse to one. The VNC client, the RDP client, the SSH client, the SFTP pair and the credential manager are all gone because the Rust binary does their jobs, plus a few they never did: live thumbnails in the dock, tabs and split panes, Wake-on-LAN for the boxes you need to wake up before you can drive them, network discovery for the ones you forgot you had. The whole client is dual licensed MIT OR Apache-2.0 and lives at [github.com/psmux/DeskVNC](https://github.com/psmux/DeskVNC).

The cross-platform piece is what turns one tray icon into one tray icon everywhere. The same install on the Windows desktop, the Mac laptop, and the Linux box. The same saved-host library follows the user through whatever combination of machines the day asks for. The next time someone asks which client to use for a new box on the estate, the answer is the binary you already have.
