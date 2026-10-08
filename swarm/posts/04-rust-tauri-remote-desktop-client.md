---
title: "Building a cross-platform remote desktop client in Rust and Tauri 2, in public"
description: "Why the DeskVNC client is written in Rust on Tauri 2, how a single binary ends up hosting three protocol stacks and an MCP server, and what the architecture buys in practice."
date: 2026-10-09
tags: ["rust", "tauri", "tauri-2", "vnc", "rdp", "ssh", "remote-desktop", "build-in-public"]
---

The DeskVNC binary hosts three full remote-desktop protocol stacks plus an MCP server in one process, written in Rust on Tauri 2. Twenty-six releases in, the choice to write it in those two layers still earns its keep on the two places that mattered at the start: protocol correctness and predictable latency on a desktop machine. The [project page](https://psmux.github.io/deskvnc/) has release notes for the same client, and the [repository](https://github.com/psmux/DeskVNC) carries the source.

This is a build-in-public look at what that stack looks like from the inside, what each layer is for, and why the architecture is the way it is.

## Two languages, one process

Tauri 2 splits an application into a Rust backend and a webview-rendered UI. The frontend is plain web technology rendered in the OS-provided webview, and the backend is Rust. The IPC between them is a typed command bridge. For a remote desktop client that split lines up cleanly with the work: the pixel pipeline and the protocol clients live in Rust, the window chrome and the saved-host library live in the frontend, and the two meet on a narrow, typed boundary.

The visible payoff is the binary size and the time to first pixel. No Chromium is bundled, because every platform already ships a webview. macOS uses WKWebView, Windows uses WebView2, and Linux uses WebKitGTK. The Rust binary itself does the heavy lifting, and native dialogs handle the file pickers and credential prompts.

## Why Rust for the protocol clients

VNC/RFB, RDP and SSH are not friendly protocols. Each has a handshake, a state machine, an extension surface, and a long tail of authentication variants. The DeskVNC VNC stack speaks RFB 3.3 through 3.8 and handles VeNCrypt, RA2 and Apple authentication in the same binary. RDP has its own connection sequence, a glyph cache, a clipboard channel and a sound channel. SSH is more orderly on the wire but inherits a sprawl of key-exchange methods, host-key algorithms, and cipher suites that all need to live in the same negotiation logic.

Writing those stacks in a language without sum types means a lot of `if state == STATE_X and feature_X_enabled(...)` checks, or a parallel hierarchy of classes that does not compile-time prevent a missed branch. Writing them in Rust means the type system carries the protocol shape. A `RfbState` enum with `Init`, `Security`, `AfterSecurity`, `Normal` variants is checked at every transition by the compiler. The same trick makes the RDP connection sequence readable, and it makes the SSH transport parameters honest.

The other payoff is latency. A remote desktop client paints frames in real time, which means no GC pauses, no JIT warm-up, no surprise. Rust's allocator and task model let the protocol code own its own runtimes, and the result is the latency numbers in the README. `dvv_screen` at scale 0.25 is 25 ms because the path from receiving a frame buffer to handing pixels back to the frontend does not pause for anything.

## The MCP server rides on the same backend

The same Rust process that drives the protocol clients also hosts the MCP server. It is named `dvv`, it speaks the same JSON-RPC surface any MCP client expects, and the tools it exposes map one-to-one onto what the human sees in the window. There is no IPC hop between the UI and the server, because they are the same process.

That structural fact is what makes the lease and generation story from the MCP post possible. The server does not have to reconcile a separate process for input arbitration; it already holds the screen buffer and the input queue by virtue of running the client. Lease invalidation on human takeover is a window-focus event in the same binary. Generation fencing is a counter next to the buffer the binary already owns.

A copy-pasteable MCP call against this backend, run on whatever transport the agent uses:

```json
{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}

{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{
  "name":"dvv_open",
  "arguments":{"machine":"design-srv-01",
               "protocol":"rdp",
               "user":"ops"}
}}

{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{
  "name":"dvv_screen",
  "arguments":{"lease":"L-7f4a","scale":0.25,"format":"png"}
}}
```

The first call enumerates tools, the second opens a saved host, the third captures the current frame. The agent on the other end can run dozens of these per second against the same lease, and a second agent on a second machine runs an independent loop with its own lease on its own protocol. The MCP server itself does not need a database, because the saved-host library is the same library the UI uses.

## What the native feel actually costs

A "native" desktop client does not ship native widgets. It ships the right behaviours for the platform: the right keychain, the right file dialog, the right window chrome, the right menu layout. Tauri gives Rust access to all of that through plugins and through direct system calls where the plugin gets in the way.

Credential storage is the most visible example. DeskVNC uses the operating system keychain on every platform: Credential Manager on Windows, Keychain Access on macOS, the Secret Service on Linux. The code is a small Rust module on each platform, and it is the difference between a settings file with secrets in plaintext and a binary that integrates with whatever credential management the user already runs. It also lets the same saved host work for the human and the agent, because there is exactly one credential store in the picture.

Network discovery, Wake-on-LAN, clipboard integration and bidirectional SFTP are the same story: small Rust modules that call the platform directly, exposed to the frontend over typed commands. The frontend never has to bundle a native library to get access.

## What twenty-six releases learned

The build is dual licensed MIT OR Apache-2.0, [hosted on GitHub](https://github.com/psmux/DeskVNC), and has 70 stars and 836 downloads across the twenty-six tagged releases as of writing. The changelog across those releases is mostly protocol details, keychain fixes, MCP tool additions and the smaller pieces that come from sharing three protocol stacks in one binary. The architecture that came out the other side is the one the project description now describes: one client for every machine you look after.

The next post in this run looks at the safety story around that client. The architecture is what makes the rest of it possible, but the architecture is not the part that lets the human trust the loop.
