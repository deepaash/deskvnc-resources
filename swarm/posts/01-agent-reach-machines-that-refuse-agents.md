---
title: "When a remote machine refuses your agent, speak its protocol instead"
description: "Citrix, VDI, jump hosts and client-owned laptops all reject the installed-agent approach to remote automation. A protocol those machines already speak gets in anyway."
date: 2026-10-09
tags: ["remote-desktop", "ai-agents", "mcp", "vnc", "rdp", "ssh", "citrix"]
---

The maintenance contract at one of our reference sites lists eighty machines across a mid-size MSP. Thirty of them are Citrix Virtual Apps. Twenty are jump hosts the team uses to reach the rest of a client's network. Thirty more are laptops the client owns outright, and a separate clause in the master services agreement forbids installing any third-party agent on them. The handful left are the MSP's own Linux servers and Mac workstations.

That estate is the honest test of "agentic" remote automation, and it is exactly where the standard story breaks down.

## Why the installed-agent approach stops at the perimeter

The funded tools that let an AI agent drive a Windows desktop all do the same thing first: they install an agent on that desktop. A small background service that captures the screen, listens for input events and forwards both to a control plane. It works, and it works well, until you point it at a machine you cannot install anything on.

Citrix and other VDI estates are the obvious wall. The session you see is a published resource on a farm; the OS image is rebuilt from a gold master every reboot, so anything an installer dropped last night is gone by morning. Anything that asks for local admin rights gets refused by group policy.

Jump hosts are a quieter wall. By definition, they sit between you and the rest of a network, and the operational rule is that nothing else runs there. Installing a new service turns the gatekeeper into a tenant, and very few teams will let you.

Client-owned machines are the contractual wall. The data owner has signed off on a remote support tool that uses only standard protocols, and that signature is what lets you in at all. A new background service is a renegotiation.

The README of [DeskVNC](https://github.com/psmux/DeskVNC) puts it in one sentence:

> The funded tools that let an agent use a Windows desktop all install an agent on that desktop, which is refused on Citrix, on VDI, on jump hosts and on anything a client owns. A protocol those machines already speak gets in anyway.

The "protocol" half of that sentence is doing the work. VNC/RFB and RDP are the same protocols those estates already open by design. RDP is what IT publishes to give helpdesk access. VNC is the de facto mirror on Linux and Mac. SSH is the workhorse for the server half of the estate. None of those require a fresh agent, because the listener has been running on the box since before any of us got involved.

## What an agent-driven session looks like in practice

The agent lives on your laptop, where you already control what is installed. From there it talks to an MCP server that sits inside the same client you would use by hand. The client owns the connection to the remote; the agent just decides what to type and where to click. The lease and generation mechanism keeps the click honest: each click carries a generation read from the most recent screen, so a click computed against a stale frame is refused before it ever reaches the wire.

A realistic four-call loop over the Citrix machine in the scenario above looks like this:

```json
{"jsonrpc":"2.0","method":"tools/call","params":{
  "name":"dvv_open",
  "arguments":{"machine":"citrix-farm-04","protocol":"rdp","user":"svc-helpdesk"}
}}

{"jsonrpc":"2.0","method":"tools/call","params":{
  "name":"dvv_control",
  "arguments":{"lease":"L-a1b2c3","mode":"exclusive","timeout":"30s"}
}}

{"jsonrpc":"2.0","method":"tools/call","params":{
  "name":"dvv_screen",
  "arguments":{"lease":"L-a1b2c3","scale":0.5}
}}

{"jsonrpc":"2.0","method":"tools/call","params":{
  "name":"dvv_click",
  "arguments":{"lease":"L-a1b2c3","x":412,"y":188,"generation":42,"button":"left"}
}}
```

The first call opens the saved host, the second takes an exclusive lease so no other input races in, the third captures the frame the agent is reasoning about, the fourth lands the click on the exact generation the agent saw. A few milliseconds after that the agent captures another frame and decides what to do next. Round-trip for the whole loop, measured against the same shape of target, is in the low tens of milliseconds on a LAN. The MCP server itself is named `dvv` and ships with the [DeskVNC client](https://psmux.github.io/deskvnc/), which is a Rust and Tauri 2 app that runs on Windows, macOS and Linux and is dual licensed MIT OR Apache-2.0.

## Why this changes the conversation

Talking about "the agent on the remote machine" forces you to ask permission each time you want to add a new box. The protocol story changes the negotiation: if the remote already accepts RDP or VNC or SSH, the answer is "good, then we already have what we need", and the agent loop on the operator side is the only thing that has to grow.

That fits the maintenance contract in the opening scenario today. The Citrix hosts already publish RDP, so `dvv_open` connects without a footprint on the farm. The jump hosts already accept SSH, so the same loop drives the terminal half of the estate. The client-owned laptops already have RealVNC or built-in macOS screen sharing on them; the contract was written around those tools, so the agent uses them. Nothing new was installed on any of them, and the work was automated end to end.

The agents still live next to you, where you can see them and stop them, which is its own story and worth its own post. The point here is the perimeter: a tool that speaks what the machine already speaks gets through doors that an installer cannot.

The repository at [github.com/psmux/DeskVNC](https://github.com/psmux/DeskVNC) has the client, the MCP server source, and 836 downloads worth of release history across twenty-six tagged builds as of writing. The protocol half of the agent story is the part that changed the answer to "is this estate automatable".
