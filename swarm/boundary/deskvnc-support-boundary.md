# DeskVNC Support and Boundary support

DeskVNC ships two complementary pieces for remote help: a stand-alone app
called **DeskVNC Support**, intended for the person who needs help, and a
**Boundary support** flow built into the main DeskVNC client, intended for the
helper. Together they let one person reach another person's machine across
networks neither side fully controls, with explicit consent at the other end and
no account for either party.

## What DeskVNC Support is

DeskVNC Support is a small companion application. It is built from the same
codebase as the main DeskVNC client and is downloaded from the same release
page. It is what you hand to the person who needs help so they can generate an
invitation and approve the session from their own keyboard. The helper stays
inside their own copy of DeskVNC; the recipient stays inside DeskVNC Support;
neither side signs up for anything.

This is the part most remote help tools do not offer. A support worker does not
need to send credentials to a hosted relay service, and the customer does not
need to install anything more than a single signed app.

## What Boundary support does inside DeskVNC

Boundary support is a connection mode inside the main DeskVNC client. The
helper opens Boundary support, pastes the invitation or the Boundary code that
the recipient generated, and DeskVNC handles the rest. The helper uses the
same DeskVNC they already use for their own machines, with one extra entry
point that takes them straight to the other side.

## Two ways to hand off a session

The Boundary support flow has two handoff styles. Both end with the recipient
approving the connection. The right one depends on whether a Boundary code
service is configured.

**Full invitation flow.** The recipient opens DeskVNC Support and presses
**Get a code** when a code service is configured, or **Create invitation** when
no code service is configured. DeskVNC Support produces a single-use
invitation. The helper pastes that invitation into Boundary support in DeskVNC
and connects. The helper cannot connect without the recipient's consent, and
the recipient sees a clear prompt to approve the connection before anything
happens.

**Shorter code handoff.** When the shared Boundary code service, or a private
Boundary code service the helper runs themselves, is configured, DeskVNC Support
can display a Boundary code built from twelve digits in four groups, for
example `1234 5678 9012`. The helper pastes the code into Boundary support in
DeskVNC and connects. The code expires after one lookup, which is what makes
it safe to read over the phone, in chat, or in email. Even when the code is
used, the person at the remote computer still approves the session from their
own keyboard, so a code on its own is never enough to start a session.

Both styles end with the same approval step on the recipient's machine. Both
styles are picked from the same DeskVNC Support screen.

## Direct encrypted path, with relay fallback

When the helper connects, Boundary support first attempts a direct encrypted
path between the two machines. Direct is preferred because it keeps traffic
end to end and lets both sides benefit from local network performance. When the
networks do not allow a direct path, such as when one or both sides sit behind
strict NAT or a firewall that blocks inbound traffic, Boundary support uses its
own relay as a fallback so the helper can still reach the recipient. The
recipient always sees a clear consent prompt regardless of which path was
chosen for the bytes.

## The recipient stays in control

The recipient controls the session at every moment it is open:

- The recipient approves the helper's connection at the moment it begins. No
  approval, no connection.
- The recipient can revoke control of mouse and keyboard at any time without
  ending the session.
- The recipient can end the session outright at any time.

These choices are made on the recipient's machine through DeskVNC Support, so
the helper never has a way around them.

## Permissions on macOS

On macOS the recipient sees clear requests for the operating system
permissions DeskVNC Support actually needs to do its job. Screen Recording and
Accessibility permissions are requested on macOS only when sharing or control
needs them. DeskVNC Support does not request permissions for capabilities it is
not using, and the requests are tied to specific actions rather than to launch
time alone.

## Where DeskVNC Support runs

DeskVNC Support ships today for the three platforms the rest of DeskVNC
supports:

- **macOS:** signed and notarized.
- **Windows:** signed.
- **Linux:** an x86_64 binary distributed in a tarball.

The signing posture on macOS and Windows means the recipient can install and
run the app without disabling operating system protections, and the tarball on
Linux is straightforward to unpack and run on a standard x86_64 install.

## Putting it together

A typical helper run looks like this:

1. Send DeskVNC Support to the person who needs help.
2. Have them open it. If a code service is set up, they press **Get a code**;
   otherwise they press **Create invitation**.
3. They read the invitation or the four-group Boundary code aloud (or paste it
   into chat), and approve the helper's connection when prompted.
4. The helper pastes the invitation or the code into Boundary support in their
   own copy of DeskVNC and connects.
5. The helper works the session until the recipient revokes control or ends
   the session.

No account is required for an attended support session on either side. Every
connection is consented to by the person being supported. That posture is what
makes DeskVNC Support and Boundary support a fit for environments where trust
is the point of the tool, not an afterthought.
