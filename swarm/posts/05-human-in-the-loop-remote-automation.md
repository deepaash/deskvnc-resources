---
title: "Human-in-the-loop is a property of the lease, not a feature flag"
description: "How a per-machine lease, a generation counter and a shared window give the operator real authority over an AI agent driving a remote desktop."
date: 2026-10-09
tags: ["ai-agents", "safety", "mcp", "remote-desktop", "human-in-the-loop", "rust"]
---

The first thing the README of [DeskVNC](https://github.com/psmux/DeskVNC) says about its MCP server is structural: it lets an AI agent open one of your saved machines, observe the screen and send clicks and keystrokes over the ordinary VNC, RDP or SSH connection, with nothing installed on the remote machine, and a person can take control back at any moment.

Every clause in that sentence is doing work, but "a person can take control back at any moment" is the one that decides whether you let an agent touch real machines. This post is about how that property falls out of the design, and what it looks like from the operator's side when it has to.

## The mental model: same process, same window

The MCP server runs inside the same binary as the UI. The agent calls into it over whatever transport MCP uses, and the tool handlers run on the same thread pool that paints frames for the window the human has open in front of them. There is no second process to coordinate, no IPC to bridge, and no separate screen buffer to keep in sync.

That shared-process shape is what makes the lease story honest. The lease is not a token negotiated with some out-of-band authority; it is an entry in a table owned by the same code that owns the input queue. When a person clicks the window, types a key, or grabs the focus, the input event reaches the same loop the agent's tools reach, and the lease on the active machine is invalidated in the same lock-free update that delivers the keystroke to the window.

The agent that was driving the machine gets the news on its next call. The shape of that call, copy-pasteable:

```json
{"jsonrpc":"2.0","id":42,"method":"tools/call","params":{
  "name":"dvv_open",
  "arguments":{"machine":"design-srv-01","protocol":"rdp","user":"ops"}
}}

{"jsonrpc":"2.0","id":43,"method":"tools/call","params":{
  "name":"dvv_control",
  "arguments":{"lease":"L-7f4a","mode":"exclusive","timeout":"60s"}
}}

{"jsonrpc":"2.0","id":44,"method":"tools/call","params":{
  "name":"dvv_click",
  "arguments":{"lease":"L-7f4a","x":317,"y":204,
                "button":"left","generation":9127}
}}

{"jsonrpc":"2.0","id":45,"method":"tools/call","params":{
  "name":"dvv_click",
  "arguments":{"lease":"L-7f4a","x":412,"y":188,
                "button":"left","generation":9128}
}}
```

The first call opens the saved machine and returns a lease handle. The second takes an exclusive input lease with a sixty-second timeout the server renews while the agent is making progress. The third call is the kind of click a model commonly gets wrong: the agent reasoned about a button at (317, 204), and the click lands. The fourth call is where the safety story shows up.

## What takeover actually returns to the agent

Suppose a human comes back to the desk in the middle of this loop, focuses the window, and clicks somewhere else on the screen. From the agent's perspective, nothing changes in the protocol stream, but the lease `L-7f4a` has been invalidated. The fourth `dvv_click` does not land and does not time out. It comes back as a structured error:

```json
{
  "ok": false,
  "error": "LEASE_LOST",
  "lease": "L-7f4a",
  "reason": "human_input_on_window",
  "new_generation": 9131
}
```

`LEASE_LOST` is a clean signal the agent can act on. It is not an exception, not a hang, and not a silent swallow. The error carries the reason ("human_input_on_window") and the current generation, which means the agent can branch: stop and ask, capture a fresh frame, or hand the task back to the human entirely. A well-built agent treats `LEASE_LOST` the same way a polite person treats a coworker saying "I'll take that one".

The lease timeout plays the same role in another case. If the agent stalls, the timeout fires and the plane invalidates the lease on its own. The next input call is refused, and the agent can check `dvv_control` with `action: "yield_status"` to see whether a person took over or the lease simply lapsed. Both paths are the same operation underneath: the lease row is cleared, the agent hears about it in band, and the protocol state on the wire stays consistent.

## Why "per-machine" is the load-bearing word

The lease is per machine, not per agent and not per session. Every saved host has its own row in the lease table, every open session has its own input queue, and every protocol stream stands on its own. The README line about ten machines being ten independent loops is the same fact stated differently.

That shape is the isolation you want when an agent goes wrong on one machine but you still need the other nine. A misfired loop on the build box does not lock out the operator from the jump host. A hung lease on the database server does not put an exclusive hold on the QA box. Each box is its own limb, and the boundaries between them are at the same place the operator's eyes and the operator's hands already are.

The same idea protects the agent. If the operator wants to take control on machine A, the lease for A is invalidated and the agent loses only A. The leases on machines B and J, where the agent is still doing useful work, are untouched. The agent resumes as soon as the operator lets go, with the same generation fence on each machine it had before.

## Generation fencing, again, as a safety property

The generation counter from the MCP post is the other half of the safety story. It catches a click computed against a stale frame before the click reaches the wire. When a human is in the loop, the screen moves under the agent's feet on every keystroke, so the fence is what keeps the agent from acting on a world it has already lost. The combination is the design:

- a lease a person can always revoke by focusing the window, and
- a generation counter the server enforces on every input call.

Both of them are simple integers travelling next to every tool call. Neither requires a vision model, a planner, a guard model, or a separate approval flow. The server enforces both, and the protocol layer never sees a click that breaks either rule.

## Why this is the property that lets the agent touch real machines

In a normal automation tool, a human-in-the-loop story is a feature on a slide. There is a button somewhere that means "pause", and the agent keeps running until somebody clicks it, and the click is best-effort. With the lease and the generation fence in the same binary as the UI, the safety property is the same primitive as the keyboard focus. The agent is not running alongside the human; the agent is running behind the human's input, with the same lock that the input has to win.

That is why "a person can take control back at any moment" reads as a sentence with a verb. The person takes control back, the agent hears about it, and the agent stops. The next call into the MCP server returns a structured error, and the loop ends on the agent's side without damage to the machine, the protocol session, or the operator's trust.

The [DeskVNC client](https://psmux.github.io/deskvnc/) is the project page, and the [GitHub repository](https://github.com/psmux/DeskVNC) has the binary, the source and the release history. Twenty-six tagged releases and an MCP server that earns its name on the safety property it ships with.
