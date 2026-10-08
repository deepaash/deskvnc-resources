# Use case: an AI agent on jump hosts and bastions

A jump host is a single SSH hop that gives access to a network segment
that the operator cannot reach directly. The agent does not need a new
path, because the path the operator already uses is the right one: a
single ordinary SSH session. The agent opens the same session the
operator would, drives the same terminal, and runs the same commands,
all over SSH, with the agent's input going through the protocol the
host already speaks.

## The situation

You have a bastion host in front of a private network, and the bastion
is the only machine your operator machines can SSH to. From the bastion,
the operator reaches the inside of the network one hop at a time. The
machines behind the bastion do not have a public address, and the
bastion does not have an agent on it. The bastion is also exactly the
sort of machine that refuses installed software, because a bastion is
deliberately minimal.

## Why the protocol level approach is the right shape

A bastion is a server that exposes one service, SSH, and the only thing
you can rely on is that the SSH port is open and the key you bring is
acceptable. The installed agent pattern does not work here, because
nothing is being installed. The protocol pattern is exactly the right
fit, because the protocol the bastion speaks is the protocol the agent
uses.

DeskVNC's `dvv` MCP server reaches the bastion over SSH, with the same
key the operator would use, and the agent drives the same terminal. The
agent reads the screen, types commands, runs long running commands
with `dvv_run`, and reads the output back as it streams.

## The SSH terminal loop in real syntax

The agent is working a bastion that is saved as `jump-01` in the host
library. The agent wants to find out who is logged in, run a quick
health check, and tail a log file.

The first call is the same as for any other host: open it, get a limb
id, take the wheel.

```jsonc
dvv_hosts   {}
// -> [{"id": "jump-01", "protocol": "ssh", ...}]

dvv_open    {"hostId": "jump-01", "perceive": true}
// -> {"limbId": "limb-92d", "size": [80, 24], "state": "attached"}

dvv_control {"limbId": "limb-92d", "action": "acquire"}
// -> {"ok": true, "lease": "lease-44e"}
```

For an SSH host, the "screen" is a terminal buffer, and the loop uses
the terminal specific calls. `dvv_term_read` returns the current
buffer, `dvv_term_send` types into it, and `dvv_run` runs a command
that returns its output as a settled result. The block below shows the
shape of each call; the full argument reference lives in the project
integration notes at `docs/AGENTS.md` and the compiled skill that
`dvv setup` writes next to each agent.

The agent wants to know who is logged in. It sends the command, waits
for the result, and reads the answer.

```jsonc
dvv_run     {"limbId": "limb-92d", "command": "who", "timeout": 5000}
// -> {"ok": true, "stdout": "ops      pts/0        2026-10-09 09:14 (10.0.0.4)\n", "exit": 0}

dvv_term_read {"limbId": "limb-92d"}
// -> {"generation": 1, "text": "ops@jump-01:~$ who\nops      pts/0        2026-10-09 09:14 (10.0.0.4)\nops@jump-01:~$ "}
```

The agent wants to check disk space on the bastion, which is a command
that runs and returns.

```jsonc
dvv_run     {"limbId": "limb-92d", "command": "df -h /var/log", "timeout": 5000}
// -> {"ok": true, "stdout": "Filesystem      Size  Used Avail Use% Mounted on\n/dev/sda1        50G   12G   36G  25% /\n", "exit": 0}
```

The agent wants to tail a log file. That is an interactive command that
streams, so the right calls are `dvv_term_send` to type the command and
`dvv_term_read` to read the screen as new lines arrive.

```jsonc
dvv_term_send {"limbId": "limb-92d", "text": "sudo tail -f /var/log/auth.log\n", "wpm": 2000}
// -> {"ok": true, "settled": true}

dvv_term_read {"limbId": "limb-92d"}
// -> {"generation": 2, "text": "Oct  9 09:14:11 jump-01 sshd[2341]: Accepted publickey for ops from 10.0.0.4 ...\n..."}
```

The terminal reads carry a `generation`, the same way the VNC and RDP
screens do. If something large has been printed since the agent last
read, an action is fenced and the agent has to look again before it
acts. Terminal limbs are not fenced by the `SCREEN_CHANGED` rule the
desktop limbs are, because a PTY echoes what it is sent into a stream
the agent reads back.

## The key, the keychain, and the agent

The agent does not need to know the key. The key lives in the
operating system keychain, the same place it lives for a person. The
host library entry references the key by name, and `dvv_open` finds the
key from the keychain for the agent. The agent sees a limb id and a
terminal, the same way a person sees a window and a prompt.

This is the property that makes a jump host approachable from an agent.
The keychain does not move into the agent, the agent does not see the
key, and the operator does not have to copy a secret anywhere to make
the agent work.

## What the agent can do on a jump host

The same three primitives cover most of the useful work.

- `dvv_run` for commands that return, including commands that take a
  few seconds and stream progress that the agent reads on the next
  `dvv_term_read`.
- `dvv_term_send` and `dvv_term_read` for the interactive case,
  including tailing, pagers, REPLs and any command that needs a TTY
  allocated.
- `dvv_files` for SFTP transfer of files in and out of the bastion,
  through the same SSH session.

The protocol is the same SSH the operator uses. There is no second
protocol to learn, and there is no second set of credentials to manage.

## The closing summary

A jump host is the cleanest case for the protocol level approach. The
host exposes one protocol, SSH, the agent uses that protocol, and
nothing is installed on the host. The same `dvv` loop that drives a
Citrix published app and a VDI desktop drives a jump host, the only
difference is the protocol and the calls. The result is an agent that
can work the inside of a network through the same door a person uses,
with the same keychain, with the same audit trail, and with a person
able to take the wheel back at any moment.

For the broader case for the protocol level approach, see
[`comparison-why-protocol.md`](comparison-why-protocol.md). For the
Citrix and VDI case, see
[`usecase-vdi-citrix.md`](usecase-vdi-citrix.md). For the homelab
case, with Wake on LAN and network discovery in the mix, see
[`usecase-homelab.md`](usecase-homelab.md). For the per machine lease
and the human takeover story, see
[`usecase-human-takeover.md`](usecase-human-takeover.md).
