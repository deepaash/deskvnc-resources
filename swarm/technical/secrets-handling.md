# Secrets handling in DeskVNC

A remote desktop client holds credentials for every machine the
operator wants to reach, and the storage decision belongs to the same
engineering care as the protocol implementation. `DeskVNC` makes the
storage decision in three deliberate layers: the operating system
keychain on each platform, an encrypted file fallback where the
keychain is unavailable, and a test that asserts the layer above is
doing its job.

The architecture and the verification habit live in detail in
[`docs/ARCHITECTURE.md`](https://github.com/psmux/DeskVNC/blob/main/docs/ARCHITECTURE.md);
this article goes into the storage choices.

## First choice: the platform keychain

The default storage backend on each supported operating system is the
one the operating system already uses for this purpose:

- **Keychain Services** on macOS. The same database the system and
  other applications use for credentials. Items stored in the keychain
  inherit the user's permission model, including the prompt that
  surfaces when an application that has not been authorised tries to
  read them and the availability boundary when the screen is locked.
- **Credential Manager** on Windows. The native vault per user, with
  the same protection boundaries as other applications using the
  Generic Credential type.
- **Secret Service** on Linux. The freedesktop.org standard, backed by
  GNOME Keyring or KDE Wallet on the desktop, and by KWallet or
  headless-compatible implementations in environments where one is
  configured.

Storing a credential through the platform API is not the same as
storing it in an SQLite blob. The keychain applies the operating
system's per-user access policy, integrates with screen-lock behaviour,
and means the operator can revoke a credential without the client
having to know it happened. It is also the path that is portable
across protocol crates, so a single credential store covers the VNC,
RDP and SSH cores.

## Second choice: the encrypted file fallback

Headless machines, locked keyrings, or environments where the operator
deliberately does not run a desktop secret service should not block a
remote desktop client from working. The fallback is an encrypted file
under the user's profile directory, using **XChaCha20-Poly1305** for
the cipher and **Argon2id** for the key derivation.

The KDF parameters are bound into the file as additional authenticated
data. The consequence is concrete: an attacker who knows the
ciphertext and the KDF output and tries to downgrade the parameters
on the legitimate user cannot do so without breaking the AEAD
authentication, because the parameters are part of what the tag
covers. That makes the format self-describing in a security-relevant
way: the only way to read it is to use the parameters recorded in
the file itself, which the runtime treats as fixed.

The plaintext of the file is the per-host credential map, indexed by
host identifier. The host identifier is opaque to the file; the file
does not record host names, host addresses, or any metadata beyond what
is needed to map a host record to its stored credential. Even with the
file in hand, an attacker only learns that some host has some
credential; correlating that back to a named machine requires the
profile database, which is stored separately.

## The rule: secrets never touch the profile database

The third layer is a property of the host library rather than of the
storage backend. The host library is the SQLite database where the
operator keeps their saved machines: friendly names, group
membership, tags, display and input controls, connection history,
live thumbnails. The credential for a host is a separate record in
the keychain or the encrypted fallback, looked up by host identifier.

The boundary between the two stores is the subject of an explicit
test, recorded in
[`docs/ARCHITECTURE.md`](https://github.com/psmux/DeskVNC/blob/main/docs/ARCHITECTURE.md).
The test calls the save-host code path with a real credential,
commits the resulting profile, then opens the SQLite file and its
write-ahead log and asserts that nothing sensitive is present in
either. The credentials section of the architecture documentation puts
it in one sentence: secrets never touch the profile database, and a
test asserts that saving a host writes nothing sensitive into the
SQLite file or its write-ahead log.

That is the kind of claim that is only worth making when there is a
test attached to it, and it is the kind of claim that earns the rest
of the design its keep. An SQLite file is a single artefact to back
up, copy between machines, or sync to a development environment; if
it carried secrets, all three of those workflows become a leak
incident. With the boundary held, the operator can move their host
library around freely.

## What this means for the operator

Three properties are worth holding onto because they fall out of the
design without further work:

- A credential stored through the keychain follows the keychain's
  permission model. Locking the screen locks the credentials. Wiping
  the keychain wipes the credentials. The operator does not have to
  remember a second location.
- A credential stored through the encrypted file survives a headless
  boot, and the file alone is not enough to read it because the KDF
  parameters are part of what is authenticated.
- The host library can be exported, indexed, queried, copied, and
  version-controlled without exposing anything sensitive, because the
  test holds the boundary between the two stores.

The combined effect is that the operator can keep a host library
that travels (across machines, across operating systems, across a
team) and keep the credentials that travel with it where the
operating system expects credentials to live.

For the protocol-feature list, the security model, and the crate
layout that puts the storage backend behind a single interface, see
[`docs/ARCHITECTURE.md`](https://github.com/psmux/DeskVNC/blob/main/docs/ARCHITECTURE.md).
