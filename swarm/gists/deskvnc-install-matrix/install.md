# DeskVNC install and availability matrix

DeskVNC is one native desktop client for VNC, RDP and SSH, written in Rust
on Tauri 2, and it ships builds for Windows, macOS and Linux from a single
release page. The same release page also carries **DeskVNC Support**, the
zero-account attended tool you can hand to a person when you want to drive
their machine.

## Where to get the build

Latest release: <https://github.com/psmux/DeskVNC/releases/latest>

Full install notes, checksum verification and the permissions the app asks
for live in `docs/INSTALL.md` inside the repository.

## Desktop client, by platform

* **Windows.** The installer is signed. Because the certificate is new,
  SmartScreen may still ask once on a clean machine: choose **More info**,
  then **Run anyway**. Once that is done on a given machine, subsequent
  launches are quiet.
* **macOS.** Signed and notarized, so it opens normally without a
  Gatekeeper detour. Screen Recording and Accessibility permissions are
  requested on first use, when sharing or control needs them.
* **Linux.** Available as an `x86_64` package the project verifies on three
  shapes: a `deb` for system installs, an AppImage for portable use, and the
  usual tarball. Per-user installs land `dvv` at `/usr/bin/dvv`; the
  AppImage copies `dvv` to `~/.local/share/DeskVNCViewer/bin/dvv` because
  the AppImage's working folder disappears on quit.

The release page is the source of truth for the file names on a given
version. The "Agents end to end" workflow checks the macOS build, the
Linux `.deb`, and the AppImage together before any release, so those three
shapes share one verified path.

## DeskVNC Support, by platform

For helping someone at the other end of the internet, download **DeskVNC
Support** from the same release page:

* **macOS.** Signed and notarized.
* **Windows.** Signed.
* **Linux.** `x86_64` binary in a tarball.

The recipient opens the support app, presses **Get a code** (or **Create
invitation** when no code service is configured), and approves your
connection. You open **Boundary support** in DeskVNC, paste the invitation,
and connect. Boundary tries a direct encrypted path first and uses its
relay fallback when the networks do not allow a direct path. The recipient
can revoke control or end the session at any moment.

A shorter handoff is to configure the shared Boundary code service or your
own private service and share the displayed four digit groups, such as
`1234 5678 9012`. The number expires after one lookup and the person at
the remote computer still approves the session. A full invitation remains
available when no code service is configured.

## SmartScreen, what to do once

On a Windows machine that has not seen the certificate before, Microsoft
SmartScreen shows a blue warning the first time the installer runs. Two
clicks take you through:

1. Press **More info**.
2. Press **Run anyway**.

That is the only detour. Subsequent launches on that machine are silent.

## Building from source

For Linux, Windows and macOS developers with Rust on hand:

```sh
npm install --prefix ui
cargo install tauri-cli --version "^2"   # if you do not have it

cargo tauri dev      # development, with hot reload
cargo tauri build    # production bundle
```

You need Rust 1.95 or newer, Node 22 or newer, and the Tauri 2 system
dependencies for your platform.

## License

DeskVNC is dual licensed, at your option, under either of:

* Apache License, Version 2.0 (`LICENSE-APACHE` in the repo, or
  <https://www.apache.org/licenses/LICENSE-2.0>)
* MIT License (`LICENSE-MIT` in the repo, or
  <https://opensource.org/licenses/MIT>)

The same terms cover the whole project, including the MCP server that ships
inside the desktop client.

## Links

* Project: <https://github.com/psmux/DeskVNC>
* Latest release: <https://github.com/psmux/DeskVNC/releases/latest>
* Page: <https://psmux.github.io/deskvnc/>
