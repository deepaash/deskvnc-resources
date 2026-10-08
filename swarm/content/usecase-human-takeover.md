# Use case: an agent starts the work, a person takes over

Attended support is the case where a real person and an AI agent are
working the same remote machine, and the person can step in at any
moment. The reason the case is interesting is approval. A team that
would never let an autonomous agent loose on a customer machine will
let an agent start a case, because a person is in the loop and can
take the wheel the moment the case needs a human. The four rules
DeskVNC's `dvv` MCP server enforces are the property that makes the
case approvable, and this page is the mechanism, the syntax, and the
shape of the handoff.

## The situation

You run a support team that takes cases on customer machines. The
cases are not all the same shape, and the cases that matter are the
ones where the customer is on the line. A first line agent can do the
routine parts of the case: open the right tool, read the screen,
collect the diagnostic, paste the answer. A second line engineer joins
when the case is non standard. The customer is on the machine the
whole time.

The question for the team is which parts of the case the agent can
do unattended, and which parts the agent can do with a person on
standby. The question for the operator is what the handoff looks
like, and what guarantees the customer has that a human can take
over.

## The per machine lease

Every machine an agent opens is a limb with its own lease. The lease
is per machine, not per session, and the lease is the only thing that
decides who is in control of the desktop at any moment. A single
agent can hold many leases, a person can hold a lease, and the two
can swap without a reconnect.

The four rules the plane enforces are how the lease stays safe to
leave running.

1. **Attach before you act.** A limb id belongs to the connection
   that opened it, so a long lived peer holds the lease for the
   duration of the case.
2. **Coordinates are fenced.** A click computed against a stale
   screen is refused, so a resize or a reconnect cannot land the
   agent's click somewhere unintended.
3. **Typing is fenced too.** A `dvv_type` or `dvv_key` is refused
   with `SCREEN_CHANGED` if something large has repainted since the
   agent last looked, because focus moves when a window opens. There
   is no override.
4. **A person can take the wheel at any moment.** Click into the pane
   and the agent is fenced out of that session. Held keys and buttons
   are released, so a half finished drag cannot strand the desktop.
   The next time the agent acts on that limb, the call returns
   `LEASE_REVOKED`, and the documented recovery is to read the screen
   again before doing anything else.

The fourth rule is the one the support team cares about, and it is the
rule the customer can see. The moment a person clicks into the pane,
the agent is no longer in control. The moment the person clicks out,
the agent resumes from the next call.

## The handoff in real syntax

The agent has opened a customer machine as `customer-9421`, taken the
lease, looked at the screen, and is doing the routine part of the
case. A second line engineer joins the case from the same DeskVNC on
the operator's machine.

The first line agent is mid task. The desktop has a half filled form
on screen.

```jsonc
dvv_hosts   {}
// -> [{"id": "customer-9421", "protocol": "vnc", "host": "10.0.42.21", ...}]

dvv_open    {"hostId": "customer-9421", "perceive": true}
// -> {"limbId": "limb-71c", "size": [1920, 1080], "state": "attached"}

dvv_control {"limbId": "limb-71c", "action": "acquire"}
// -> {"ok": true, "lease": "lease-a23", "holder": "agent"}

dvv_screen  {"limbId": "limb-71c", "form": "full", "scale": 0.25}
// -> {"generation": 1, "png": "<base64>", "width": 480, "height": 270}
```

The agent reads the screen, decides the next step is non standard, and
hands the pane to the engineer. The handoff is the engineer clicking
into the pane, and the plane releases the held keys and buttons so the
desktop is never stranded.

The engineer clicks into the pane. The pane is the same DeskVNC
window, and the engineer sees the same screen the agent saw. The
agent's next call on that limb would return `LEASE_REVOKED` while the
engineer is in the pane. The agent checks lease status with the
documented `dvv_control` action `yield_status`, which is the call the
project skill recommends for this case.

```jsonc
dvv_control {"limbId": "limb-71c", "action": "yield_status"}
// -> {"holder": "human", "since": "2026-10-09T09:14:11Z"}
```

When the engineer finishes the non standard step, the engineer clicks
out of the pane. The agent reads the screen again, picks up the lease
on the next `dvv_control acquire`, and continues from the next call.

```jsonc
dvv_control {"limbId": "limb-71c", "action": "yield_status"}
// -> {"holder": "agent", "since": "2026-10-09T09:15:02Z"}

dvv_screen  {"limbId": "limb-71c", "form": "full", "scale": 0.25}
// -> {"generation": 2, "png": "<base64>", "width": 480, "height": 270}
```

The generation number moved, which is the agent's signal that the
screen is new. The agent's next click has to be computed against the
new generation.

## Why this is what makes agent driven support approvable

Three properties come out of the lease model that matter on a
support desk.

**A human is never locked out.** A pane that an agent holds is a pane
a person can click into. The agent is fenced out the moment the
person is in, and the desktop is never stranded. This is the property
a security team can sign off on, because the human is always in
control of the desktop. The plane surfaces the takeover to the agent
as a `LEASE_REVOKED` result on the next call, so the agent knows the
case has been handed back to a person and stops retrying blindly.

**Every machine is its own lease.** A second line engineer joining one
case does not hand the agent control of any other case. Ten cases open
at once are ten independent leases, and a person taking one pane
yields that one pane, not the rest. The support team can mix agent
led and human led cases in the same DeskVNC window.

**The agent is fenced, not paused.** The agent is not in a
cooperative state where it might do something between the person
clicking into the pane and the next call. The pane is a real hand
off. The agent's next call returns `LEASE_REVOKED`, and the refusal
is a guarantee the support team can lean on.

The combination is what an attended support case needs. The agent
does the routine parts, the person does the non standard parts, the
desktop is never without a person who can take the wheel, and the
case is closed with a full audit trail of who was in control of the
desktop at every moment.

## What this changes for the support team

The shape that opens up is one where the agent's value is not the
case it can close alone, but the case it can start. A first line
agent can open the customer's machine, take the routine steps,
collect the diagnostic, and hand the pane to a second line engineer
with the screen already read. The handoff is one pane, the screen
state is already current, the credentials are already in the
keychain, and the engineer is looking at the case the agent saw.

The same shape works the other way. A second line engineer can
hand a pane to the agent to do the routine follow up after the case
is closed, with the engineer confident the agent will stop at the
first thing the engineer wants to see.

## The closing summary

Attended support is the case where the per machine lease and the
human takeover semantics are the property the team buys. An agent
that holds a lease the moment a person clicks into the pane is an
agent the team can approve. The same `dvv` loop that drives a
Citrix published app, a VDI desktop, a jump host and a homelab
drives an attended support case. The only difference is the
human, and the only thing the human has to know is that clicking
into the pane is enough.

For the broader case for the protocol level approach, see
[`comparison-why-protocol.md`](comparison-why-protocol.md). For the
Citrix and VDI case, see
[`usecase-vdi-citrix.md`](usecase-vdi-citrix.md). For the jump host
case, with the SSH terminal calls, see
[`usecase-jump-hosts.md`](usecase-jump-hosts.md). For the homelab
case, with Wake on LAN and network discovery in the mix, see
[`usecase-homelab.md`](usecase-homelab.md).
