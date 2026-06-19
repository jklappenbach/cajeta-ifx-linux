# cajeta-ifx-linux — Spec

**Status:** Draft. Requirements for the **Linux** backend of `ifx` (window / input / audio).
This is a requirements document (the *what* and *why*); the task breakdown is the plan.

## 1. Overview & why
`dev.cajeta.ifx.linux` is the Linux realization of the portable `ifx` contract
(`cajeta.ifx`). It FFI-binds Linux system APIs and registers `WindowBackend` / `AudioBackend` /
`InputBackend` implementations through `BackendRegistry`. The app codes once against `ifx`; this
library is selected at runtime on Linux (registry + probe + priority, with `CAJETA_IFX_*`
overrides). Umbrella design: the cajeta repo's `documents/cajeta-gfx/cajeta-gfx-spec.md` §9.

## 2. Capabilities (what it binds)
- **Window + Surface:** Wayland (primary) + X11/XCB (fallback); produces the opaque `Surface` the gfx swapchain pairs with
  `VK_KHR_wayland_surface / VK_KHR_xcb_surface`.
- **Input:** libinput / evdev (+ udev) — keyboard/mouse arrive via the window event queue; gamepads via the input
  API.
- **Audio:** PipeWire (primary) + PulseAudio + ALSA — output and capture.

> **Notes.** Two display backends in one library; the runtime probe picks Wayland when WAYLAND_DISPLAY is set, else X11. Audio probes PipeWire then PulseAudio then ALSA.

## 3. Goals
- Implement the full `ifx` SPI for Linux behind the one portable API surface — no Linux concept
  leaks to callers.
- Register + `probe()`/`priority()` so the dispatcher binds it on Linux; honor `CAJETA_IFX_*`.
- Hand a correct opaque `Surface` to the gfx swapchain (`VK_KHR_wayland_surface / VK_KHR_xcb_surface`).
- Track Linux API versioning **here**, isolated from the other backends.

## 4. Non-goals
- No portable-contract changes (those live in stdlib `ifx`).
- No graphics/render code — gfx owns the swapchain; this only supplies the `Surface`.
- No UI toolkit, input action-mapping, or audio mixing/DSP (engine / Glorias concerns).
- No codecs — capture/recording is the `cajeta-ifx-harness` backend + optional `VideoSink` providers.

## 5. References
- Umbrella design: `cajeta-gfx-spec.md` §9 (platform layer — §9.1 codegen-vs-HAL, §9.3 packaging,
  §9.4 binding model).
- Contract: `cajeta.ifx` — `Backend` / `WindowBackend` / `AudioBackend` / `InputBackend`.
- Selector: the `cajeta-ifx-backend` melt (target → backend).

---

## Appendix A — Vendor SDK details (2025-2026 research)

### Binding: pure C-ABI FFI (no shim, no JNI) — but TWO stacks each domain
All libs are clean C ABIs. The defining trait: **Linux has no single blessed API** — the backend
must ship **both** windowing stacks (Wayland + X11) and **runtime-detect** the audio server.

### SDK reference
| Domain | Modern (libs/protocols) | Legacy / fallback | Deprecated | Notes |
|---|---|---|---|---|
| Window | **Wayland** `libwayland-client` + **xdg-shell** (`xdg_wm_base`/`xdg_surface`/`xdg_toplevel`); decorations via xdg-decoration / `libdecor`; FPS mouse via `zwp_relative_pointer` + `zwp_pointer_constraints` | **X11 via libxcb** (`xcb/xcb.h`) | Xlib (use xcb) | Must support **both**; SDL3 favors Wayland over X11 by default; runtime-probe both |
| Surface | `VK_KHR_wayland_surface` (`vkCreateWaylandSurfaceKHR`) | `VK_KHR_xcb_surface` / `VK_KHR_xlib_surface` | — | enable the ext matching the chosen backend |
| Input (kbd/mouse) | compositor events: `wl_keyboard`/`wl_pointer`/`wl_touch` (+ `libxkbcommon`); X11 via XCB events | — | — | `libinput` is compositor-side, not a client API |
| Gamepad | **evdev** (`/dev/input/event*`, `libevdev`) + **libudev** (hotplug); rumble via evdev FF ioctls (`EVIOCSFF`) | SDL3 gamepad DB for mappings | legacy `js*` (`linux/joystick.h`) | needs **seat / udev-ACL** access |
| Audio | **PipeWire** `libpipewire-0.3` (low-latency, capture) | **libpulse**, then **ALSA** `libasound` | — | **runtime-detect** server; PipeWire exposes Pulse/JACK/ALSA compat |

### Capability support & gaps (vs spec §9.7) — the hard ones
- **No programmatic window positioning on Wayland** → no-op; compositor owns placement.
- **No cursor warp on Wayland** → use **pointer-lock + relative motion** (the portable primitive).
- **Client-side decorations** (GNOME/Mutter ignore server-side) → be ready to self-draw / use `libdecor`.
- **evdev needs seat/udev-ACL permission** → may see no gamepads in sandboxes/containers; report, fall back.
- **Runtime backend selection mandatory** for BOTH windowing (Wayland vs X11) and audio
  (PipeWire→Pulse→ALSA) — this is exactly the `ifx` registry+probe pattern, one level down.
- **Supported:** rumble (evdev FF), low-latency audio (PipeWire quantum / ALSA period), capture
  (PipeWire + portal). Loopback/system-output capture via PipeWire+portal (partial). No exclusive HDR story.

### References
- xdg-shell: https://wayland.app/protocols/xdg-shell · SDL Wayland policy: https://wiki.libsdl.org/SDL3/README-wayland
- `VK_KHR_wayland_surface`: https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_wayland_surface.html
- evdev/gamepad: https://wiki.archlinux.org/title/Gamepad · PipeWire: https://wiki.archlinux.org/title/PipeWire

---

## Appendix B — Interop mechanism (direct C FFI + dlopen)
**No shim, no JNI.** libwayland-client, libxcb, xkbcommon, libpipewire-0.3, libpulse, libasound,
libinput, libevdev, libudev are all clean C ABIs bound through Cajeta `@Native`. Runtime backend
selection (Wayland vs X11; PipeWire→Pulse→ALSA) uses **`dlopen`/`dlsym`** so the binary only requires
the libs actually present. Wayland listeners and evdev reads are C callbacks/fds Cajeta drives. Like
Windows, the interop work is binding headers + the runtime probe, not bridging a language.
