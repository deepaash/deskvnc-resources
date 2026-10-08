# The DeskVNC Support and Boundary support security model

DeskVNC Support, paired with the Boundary support flow inside DeskVNC, is
designed around one principle: the person whose machine is being reached is
always the one who decides whether to let the session happen, and stays in
control while it is open. Everything else about the system is built to make
that principle hold in practice.

## Consent comes before any connection

Every attended support session starts with the recipient pressing a button on
their own keyboard. The recipient opens DeskVNC Support and either presses
**Get a code** when a Boundary code service is configured, or presses **Create
invitation** when no code service is configured. The helper, working from their
own copy of DeskVNC, pastes that invitation or that code into the Boundary
support view and asks to connect. The recipient then sees a clear approval
prompt on their own machine, and only acts on that prompt does a connection
open.

There is no path by which the helper can reach the recipient without that
approval. The invitation or the code alone is not sufficient. The Boundary
session is held open only as long as the recipient keeps it held open.

## Codes that expire after one lookup

A Boundary code such as `1234 5678 9012` is built from twelve digits arranged
in four groups. The code is single use. Once the helper uses the code to look
up the open invitation, the code expires, and it cannot be used again to open
another session. This is what makes the code safe to read aloud, paste into a
chat thread, or include in an email. The code is short by design, and the
expiry is what keeps it short safely.

The recipient configures the source of those codes. The default is the shared
Boundary code service that ships with the Boundary support flow, and an
organization can stand up its own private code service on its own
infrastructure instead, with the workflow documented in the repository's
`14-support-codes.md` file. Either way, a code is not a permanent credential;
it is a one-shot handoff that the recipient's machine recognizes once and
only once.

## The recipient can revoke or end the session at any time

Consent at the start of the session is not the only control the recipient
keeps. While the session is open, the recipient can revoke mouse and keyboard
control from the helper without ending the session, which means the helper
sees what is on screen without acting on it. The recipient can also end the
session outright. Both moves are made from the recipient's copy of DeskVNC
Support, never from the helper's side, so the helper cannot lock the recipient
out of their own machine.

## Direct encrypted path, with a real fallback

When the helper connects, Boundary support tries a direct encrypted path
between the two machines first. The direct path keeps traffic end to end and
preserves the latency of a local network. When the networks do not allow a
direct path, such as when one or both sides are behind NAT or a firewall that
blocks inbound traffic, Boundary support falls back to its own relay so a
session can still be established. The recipient's consent prompt is shown
regardless of which path was chosen. Boundary support's job is to make the
session possible across networks neither side fully owns, and to do so
without ever making consent optional.

## No account anywhere on the support path

An attended support session runs on consent and on nothing else. The
recipient registers nothing, enrolls in nothing, and hands over no
credentials. The helper arrives through their existing copy of DeskVNC,
with no hosted relay account to create and no tenant to provision. The
handoff is purely an invitation or a code plus an approval on the other
end, which keeps the support flow confined to a single consent moment
instead of a standing credential relationship.

## Permission requests match what is actually used

On macOS, Screen Recording and Accessibility permissions are requested only
when sharing or control needs them. The recipient is not asked for a permission
that DeskVNC Support does not currently use, and the request is tied to the
specific action that requires it rather than to a blanket prompt at launch
time. This keeps the permission grant on the recipient's terms: a permission
that is not needed for the chosen action is not requested at all.

## Distribution integrity

The DeskVNC Support application is built and signed for the platforms it
ships on: signed and notarized on macOS, signed on Windows, and distributed as
an x86_64 binary in a tarball for Linux. The signing posture means recipients
install and run the app without having to weaken their operating system
protections, and the Linux tarball is unpacked and run on a standard x86_64
install. Checksum verification of the downloaded package is documented in
`docs/INSTALL.md` in the repository, and the same verification is recommended
any time a one-shot support tool is being handed to someone.

## What this model is good for

The combination of consent-first handoff, one-shot codes, revocable control,
direct-then-relay transport, accountless design, on-demand permission requests,
and signed distribution is what makes DeskVNC Support and Boundary support a
fit for environments where trust is the point of the tool. The mechanism is
small enough to explain to a customer in a few sentences, and the recipient's
side of the flow is plain enough that the customer always knows what they are
approving.
