---
title: "Driving a remote desktop from an MCP server, with leases and generations"
description: "An engineering look at the MCP server inside DeskVNC: the four-call loop, the lease that prevents input races, the generation counter that refuses stale clicks."
date: 2026-10-09
tags: ["mcp", "model-context-protocol", "remote-desktop", "rust", "vnc", "rdp", "ai-agents"]
---

Measured against a 1920x1080 Windows desktop on a LAN, an observe-then-act cycle through the MCP server inside [DeskVNC](https://github.com/psmux/DeskVNC) takes 19 ms end to end. That is the four-call loop, server included, and it puts the tool at roughly 52 actions per second per machine. Ten machines, ten independent loops, none of them sharing a lock.

That number is what made the design feel worth writing up, because it means an MCP loop over a remote desktop is cheap enough to sit inside an ordinary agent loop without distorting it. Here is the engineering shape behind the number.

## What an MCP server is doing inside a remote desktop client

The Model Context Protocol gives a process a way to expose a JSON-RPC surface that an LLM or an agent runtime can call as if the tools were local functions. The server registers a list of tools, each with a name, an input schema and a handler. The client sends a `tools/call` request, the host runs the handler, the result comes back. The transport is usually stdio, with HTTP and WebSocket options for longer reach.

Putting the server inside the same process as the desktop client matters more than it sounds. The server can see every saved host, every protocol session it has live, and the precise state of any frame it has just rendered for the human sitting at the screen. The agent and the person are looking at the same picture, by construction.

[DeskVNC](https://psmux.github.io/deskvnc/) is a native client for VNC, RDP and SSH written in Rust on Tauri 2. It runs on Windows, macOS and Linux, holds credentials in the operating system keychain, and ships with an MCP server named `dvv`. The server has tools that mirror what the human can do, plus a few the human rarely does directly.

## The four-call loop

The pattern that covers most interactions is a tight four-call loop:

1. `dvv_open` once, to get a lease handle for the saved machine.
2. `dvv_control` to acquire an exclusive or shared input lease with a timeout.
3. `dvv_screen` to capture the current frame, scaled.
4. `dvv_click`, `dvv_type`, `dvv_key` or `dvv_scroll` to act on that frame.

After the first call, the steady-state for an agent reasoning about the screen is the last two.

Real JSON-RPC, copy-pasteable:

```json
{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{
  "name":"dvv_open",
  "arguments":{"machine":"design-srv-01","protocol":"rdp","user":"ops"}
}}

{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{
  "name":"dvv_control",
  "arguments":{"lease":"L-7f4a","mode":"exclusive","timeout":"60s"}
}}

{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{
  "name":"dvv_screen",
  "arguments":{"lease":"L-7f4a","scale":0.25,"format":"png"}
}}

{"jsonrpc":"2.0","id":4,"method":"tools/call","params":{
  "name":"dvv_click",
  "arguments":{"lease":"L-7f4a","x":317,"y":204,
                "button":"left","generation":9127}
}}
```

Three things in that exchange are worth pointing out.

The first is the lease. `dvv_open` returned a handle tagged `L-7f4a`. Every later call passes it back. The lease has a timeout, and the server resets the timer when the agent is making progress. The lease is the reason two agents cannot type into the same field at the same time, and the reason a human can grab the keyboard back by clicking the window: the human's input invalidates the agent's lease, the next `dvv_click` returns `LEASE_LOST`, and the agent knows to stop.

The second is the scale. `dvv_screen` at scale 0.25 against a 1920x1080 source is the number from the README: 25 ms to capture, encode, ship. That is the cost the agent pays per observe step, and it is the cost that keeps the loop inside the agent's own thinking time.

The third is the generation counter. Each `dvv_screen` response carries a monotonic integer tied to that lease. Every input tool then requires you to pass the generation the click was computed against. If the screen has moved on since that generation, the server refuses the call rather than letting the click land on a world that no longer exists.

## Why the generation counter is the actual safety property

The agent loop is, at heart, a sense-think-act cycle. The act half is dangerous if it is computed against a world that no longer exists. A button that was visible in the frame the agent reasoned about can be gone by the time the click arrives, the menu can have collapsed, a dialog can have appeared in front. With a generation fence, the click is refused before it lands. The agent reads the screen again and re-decides on a frame that reflects the current state.

This is the bit that turns the four-call loop from "drive the cursor" into "drive the cursor against a state machine the agent can trust". The fence does not need a vision model, a planner, or a guard model. The lease and the generation are two integers that travel next to every tool call, and the server enforces both before the click ever reaches the protocol layer.

## The numbers and what they leave you

The README measurements, captured on the LAN setup above, are the steady-state numbers for the loop: `dvv_open` and attach in 4 ms, `dvv_control` acquire under 1 ms, `dvv_screen` at scale 0.25 in 25 ms, the full observe-then-act cycle in 19 ms (about 52 actions per second), and `dvv_type` at 447 characters per second. Each machine is its own limb, so ten machines give you ten loops, not one contended for ten.

What that buys the agent is headroom. At 19 ms a cycle the agent has plenty of time to think between actions without the screen seeming laggy, and it can sustain burst typing at near half a kilocharacter a second without losing frames. What it buys the operator is the rest of the safety story: a per-machine lease a person can always reclaim, and a generation counter that catches stale reasoning before it lands on the wire. Together they are the reason an MCP server inside a remote desktop client is worth keeping on the same desk as the agent that calls it.

The source and a working client are at [github.com/psmux/DeskVNC](https://github.com/psmux/DeskVNC).
