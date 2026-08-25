# InnovaGuard: QUARANTINE gets a real security daemon

We've been heads-down on a major architecture shift for QUARANTINE, InnovaGuard's
kernel-level data flow enforcement tool for Linux. Here's what changed, what's
actually working, and what's still on the roadmap. No marketing spin, if
something isn't built yet we say so.

## The short version

QUARANTINE started as a Python + Tkinter application talking to Linux's
`fanotify` API directly. It worked, and it's still there. But a Python GUI
process is the wrong place to put a security enforcement engine, so we split
it apart:

```
React + TypeScript (desktop GUI, via Tauri)
        |
   Unix socket, authenticated per-connection
        |
Rust security daemon (runs as root)
        |
   fanotify (kernel)     SQLite (event history + policy)
```

The daemon is the only privileged component. The GUI never touches the
kernel and never runs as root, it just sends JSON commands over a socket
that checks the real UID of whoever's connecting before honoring anything
that changes protection state.

## What's actually built and tested

Not scaffolding, not mocks. Things we ran against a live kernel and verified:

- **Real fanotify enforcement in Rust**, ported from the original Python
  implementation, marking directory trees recursively and denying `open()`
  calls at the syscall level, before the requesting process gets the file
  handle.
- **SQLite-backed event history and policy storage**, replacing the old
  JSON-file-only approach. Applications, boundaries, trust list, quarantine
  state, and every allow/deny decision all persist across restarts now.
- **A security score computed from real configuration**, not a fixed number.
  It changes when you actually add protected directories, set boundaries,
  or quarantine something.
- **Auto-quarantine, trusted-app bypass, and zero-trust default-deny**, all
  carried over from the Python version and re-verified against the daemon.
- **7 unit tests** covering tag matching, boundary evaluation,
  quarantine/release, trust, and event counting, run against in-memory
  SQLite so nothing touches your real filesystem during tests.
- **A Python CLI (`innova`) and client library** that talk to the daemon over
  the same socket protocol, so scripting and automation aren't locked out
  of the new backend.
- **A React + TypeScript desktop GUI** (Tauri 2), with a live Dashboard and
  Applications page, both polling the daemon and rendering real state, not
  placeholder data.

## What's not done yet

- Only two GUI pages exist so far: Dashboard and Applications. The daemon
  already supports what a Sensitive Data page, Quarantine Center, and Rules
  builder would need, they just haven't been written.
- No scan engine, no process ancestry view, no network monitoring, no
  firewall module, no system tray, no notifications, no onboarding flow yet.
- **eBPF was deliberately skipped.** Real eBPF needs a verifier feedback
  loop against a specific kernel to get right, and shipping kernel bytecode
  that hasn't been tested against your actual kernel is a bad trade. We're
  staying on fanotify (reached from Rust via safe FFI) until there's a real
  plan to test eBPF properly.
- **Windows and macOS aren't supported, and won't be through a simple
  porting effort.** `fanotify` is a Linux-only kernel API. Supporting other
  platforms would mean separate backend implementations using each OS's own
  security APIs (EndpointSecurity on macOS, a minifilter driver on Windows),
  which is a different project, not a build flag.

## Known issues if you're building it yourself

If you're compiling the Tauri GUI on Linux, a few real problems came up
during testing that are worth knowing about upfront:

- **`failed to run linuxdeploy` during AppImage bundling** usually means
  FUSE isn't installed (`fuse2` on most distros). If installing it doesn't
  help, `NO_STRIP=1 APPIMAGE_EXTRACT_AND_RUN=1 npm run tauri build` works
  around it.
- **White screen on launch**, often with `Failed to create GBM buffer` in
  the terminal, is WebKitGTK failing to hardware-accelerate against certain
  GPU drivers. Fix with
  `WEBKIT_DISABLE_COMPOSITING_MODE=1 WEBKIT_DISABLE_DMABUF_RENDERER=1`
  before launching.
- **Build fails with an `edition2024` Cargo error** means your Rust
  toolchain is too old, almost always because it came from a distro package
  instead of `rustup`. Needs 1.85+.

Full install steps per distro (Arch, Debian/Ubuntu, Fedora) and the full
troubleshooting writeups are in the README.

## Why this matters

The old architecture worked, but it meant the thing responsible for
deciding whether to let an application touch your SSH keys was living in
the same process as the settings dialog. Splitting enforcement into its own
privileged daemon, with a narrow authenticated command surface, is the
correct shape for a security tool even if it's more work upfront. The
Python implementation isn't going anywhere, it's still fully functional
standalone, but new development is happening on the Rust/React stack from
here.

As always: the local policy engine stays open source, nothing phones home,
and if we can't demonstrate a feature actually working against a real
kernel, we don't claim it works.
