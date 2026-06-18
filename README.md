# cajeta-ifx-linux

`dev.cajeta.ifx.linux` — the **Linux** backend for **ifx**, the Cajeta interface-framework
facade for window / input / audio. It implements the `ifx` backend SPI by FFI-binding Linux system
APIs and registers itself so the `ifx` dispatcher selects it at runtime on Linux.

Part of the Cajeta graphics stack: stdlib **`ifx`** owns the portable, write-once contract; this
repo is one **optional, vendor-versioned backend**, pulled in (target-conditionally) via the
**`cajeta-ifx-backend`** melt. App code never imports this directly — it codes against `ifx`.

## What it binds
- **Window / surface:** Wayland (primary) + X11/XCB (fallback) → `VK_KHR_wayland_surface / VK_KHR_xcb_surface`
- **Audio:** PipeWire (primary) + PulseAudio + ALSA
- **Input:** libinput / evdev (+ udev)

> **Notes.** Two display backends in one library; the runtime probe picks Wayland when WAYLAND_DISPLAY is set, else X11. Audio probes PipeWire then PulseAudio then ALSA.

## Status
**v0.1 — library skeleton** (cajeta build-tool project). No implementation yet — see
[`documents/`](documents/) for the spec and plan.

## Build
```sh
cajeta build      # emits the .cja library
cajeta test       # unit tests
```

## License
MIT © 2026 Julian Klappenbach. See [LICENSE](LICENSE).
