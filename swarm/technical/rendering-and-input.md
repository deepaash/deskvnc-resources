# Rendering and input in DeskVNC

The renderer and the input path are where the engineering substance of a
remote desktop client pays off in a way the user can see. `DeskVNC`
treats both halves as carefully designed surfaces: how a frame reaches
the screen, and how local input reaches the remote machine in the
shape the user expects. This article covers both.

## The renderer: WebGL2 with no full-frame IPC

A remote frame is decoded in the Rust protocol core, which means heavy
pixel work happens outside the browser engine, then painted through WebGL2
in the React UI. The reason that matters is the kind of message that
crosses the Tauri IPC boundary: not a complete framebuffer, but a stream of
binary dirty-rectangle messages. Each message carries the rectangle that
changed and the pixel payload for it, and the webview uploads that payload
directly into a single WebGL2 texture. The result is that no full
framebuffer ever has to be serialised as JSON, base64 encoded, or copied
through a normal DOM path. On a 1920x1080 desktop that saves seconds per
minute of motion, and more importantly removes the obvious stutter that
appears the moment a different layout or any other per-pixel overhead
creeps in.

The crate layout reinforces this division: `vnc-core` and `remote-pixel`
own the protocol-side work, and the React side owns everything from
texture upload onward. Pixels are produced in one place, and consumed in
exactly one other place. There is no second copy for caching, and no
serialised frame to ship to a worker. The whole path has one hot edge,
which is what makes the project easy to profile.

### Why WebGL2 specifically

A few alternatives were available when this renderer was first written. A
plain `<canvas>` 2D context is slow for the long, full-screen write inner
loop that VNC and RDP run, and it copy-blits when the texture and the
display diverge, which is the normal case. WebGPU would have been the
modern choice, except the platform webviews on the three target operating
systems are at different points in their WebGPU support, and the cost of
holding back the lowest common denominator would land on the user. WebGL2
is the boundary that everything exposes, with stable binary texture upload,
hardware-accelerated blitting, and a long history of practical use. It
also gives a single, simple surface for the H.264 path to land on, because
both macOS, Windows, and Linux webviews expose WebCodecs `VideoFrame`
objects whose backing buffers can be composited straight into the same
texture.

### H.264 hardware acceleration

The VNC core handles H.264 framing and the bookkeeping required to keep
the decoder context consistent across frames. The decode itself runs in
the webview through WebCodecs, and the GPU path comes from the webview's
own implementation: VideoToolbox on macOS, D3D11 on Windows, and VAAPI on
Linux. The benefit is that H.264 does not require DeskVNC to ship its
own decoder, to license a third party codec, or to special case the three
operating systems individually; the webview already knows the local
acceleration story, and it exposes one surface to the application.
Combined with the dirty-rect upload, the H.264 path runs at the speed of
whatever the local GPU gives the webview, with no Rust-side copy in the
bottleneck.

## Input that matches what the user does

The input path is the half most remote clients get wrong in subtle ways.
DeskVNC's design treats two things as not negotiable: scrolling has to
follow the system direction setting, and the typing pipeline has to carry
keyboard layouts and input methods across unchanged.

### Scroll and trackpad direction

Scroll wheel events and trackpad gestures arrive at the remote machine in
the same physical orientation the user sees on their own screen,
regardless of the system setting for natural scrolling on the local
machine. The protocol core applies the conversion before the event is
shipped, which is the only correct place to do it: the conversion is a
property of the user, not of the mouse. A scroll that is "up" to the user
on this machine has to be "up" on the remote machine, and natural or
reverse on the local system should not change that. The same code path
covers high-resolution trackpads, where the raw event carries a precise
scrolling delta that would be lost if it were collapsed to discrete wheel
"clicks".

### Keyboard layouts, dead keys, and CJK input methods

The keyboard path ships the user's actual keystrokes, including the
layout-aware character that the local input method produces, rather than
the underlying keysym or scancode. That is what makes a German user
typing `ß` on a US-keyboard remote, or a Japanese user moving through
hiragana with an IME, work the way they expect. The QEMU scancode
extension is used in scancode mode when the server advertises it, which
gives a per-keystroke path that survives layout drift between client and
server; layout-aware text input runs alongside, because some inputs
(particularly IME composition) need the local pipeline. Dead keys are
preserved through the same pipeline: the local input method sees the
sequence, the remote sees the composed character. For CJK, composition is
handled in the webview on the local side, and the committed string is
what crosses the protocol. The result is that a remote Windows desktop
with a Japanese IME sees exactly the same input as a local one.

### Three-tier shortcut pass-through

A remote desktop is also a place where the local machine has its own
shortcuts. DeskVNC carries a three-tier pass-through model so the user
chooses, per shortcut, whether it stays local, goes to the remote, or
gets a configurable interpretation on both sides. The same tier model is
what lets the agent plane toggle shortcuts cleanly: a `meta+r` issued by an
agent can be made to always cross to the remote, while the local user's
own `meta+r` can stay local. Global capture uses the platform API path
and needs Accessibility permission on macOS, which is the same permission
boundary every global keyboard hook on the platform needs.

## What this combines to

The renderer and the input path are connected by one invariant: no full
framebuffer ever crosses IPC, and no keystroke ever hits the protocol
until it is the keystroke the user expected. That is what makes a 19
millisecond observe-then-act loop realistic, because the only places data
copies are the texture upload (one short pipeline stage) and the input
encode (a small event). The full design lives in
[`docs/ARCHITECTURE.md`](https://github.com/psmux/DeskVNC/blob/main/docs/ARCHITECTURE.md),
and the user-visible behaviour is described in the
[project README](https://github.com/psmux/DeskVNC).
