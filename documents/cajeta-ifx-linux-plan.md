# cajeta-ifx-linux — Plan

Derived from `cajeta-ifx-linux-spec.md`. Outline numbering `1` / `a` / `1`.
Checkbox legend: `[x]` done · `[~]` partial · `[ ]` not started.

## 1. Window + Surface backend
Linux window creation via Wayland (primary) + X11/XCB (fallback), producing the opaque `Surface` for `VK_KHR_wayland_surface / VK_KHR_xcb_surface`.

   **TDD**
   a. [ ] `probe()` returns true under Linux, false elsewhere (env-forced).
   b. [ ] `createWindow` + `surfaceOf` yield a non-headless `Surface` with the requested extent.

   **Deliverables**
   a. [ ] `LinuxWindowBackend implements WindowBackend` + self-registration.
   b. [ ] FFI bindings for Wayland (primary) + X11/XCB (fallback) (create / destroy / poll) + the surface handle.

   **Acceptance Criteria**
   a. [ ] A window opens on Linux and the gfx swapchain presents to its `Surface`.

## 2. Input
   **TDD**
   a. [ ] Synthetic libinput / evdev (+ udev) events surface as `WindowEvent` / `InputDevice` state.

   **Deliverables**
   a. [ ] keyboard/mouse via the window queue; gamepads via libinput / evdev (+ udev).

   **Acceptance Criteria**
   a. [ ] Input round-trips on a real Linux device.

## 3. Audio
   **TDD**
   a. [ ] An output stream plays a known buffer; capture reads a known input.

   **Deliverables**
   a. [ ] `AudioBackend` over PipeWire (primary) + PulseAudio + ALSA (output + capture).

   **Acceptance Criteria**
   a. [ ] Audio output is audible / captured PCM matches a reference within tolerance.

## 4. Registration & dispatch
   **TDD**
   a. [ ] Registers at load; the dispatcher selects it on Linux; `CAJETA_IFX_*` override works.

   **Deliverables**
   a. [ ] static-init registration into `BackendRegistry`; `priority()` == 100.

   **Acceptance Criteria**
   a. [ ] On Linux with this library linked, `ifx` binds it with no app changes.

## Dependencies & sequencing
- Gated on the stdlib `cajeta.ifx` contract landing (`runtime/src/cajeta/ifx/`).
- The Surface/WSI hand-off is gated on the gfx swapchain (`cajeta.gpu.gfx`, spec Part GP-1 §4.2).
- Versioned independently of the other backends (own Linux API cadence).

---

## Appendix A — Binding & capability-gap work (from vendor research)

### 5. Binding (pure C FFI, dual-stack)
   **TDD**
   a. [ ] Runtime probe selects Wayland by default (SDL3 policy), else X11 (env override).
   b. [ ] Audio probe selects PipeWire → PulseAudio → ALSA.

   **Deliverables**
   a. [ ] **Both** windowing backends: Wayland (`libwayland-client` + xdg-shell) **and** X11 (libxcb).
   b. [ ] Audio multi-backend (libpipewire-0.3 / libpulse / libasound) with runtime detection.
   c. [ ] evdev + libudev gamepad (hotplug, rumble via FF ioctls); seat/udev-ACL permission handling.

   **Acceptance Criteria**
   a. [ ] One binary runs on Wayland and X11 sessions and on PipeWire/Pulse/ALSA hosts.

### 6. Capability gaps & fallbacks (vs spec §9.7)
   **Deliverables**
   a. [ ] Wayland: **no window positioning** (no-op), **no cursor warp** → pointer-lock + relative
      motion; client-side decorations (self-draw / `libdecor`).
   b. [ ] `supports()` reflects the active stack (e.g. positioning false on Wayland, true on X11).
   c. [ ] gamepad-absent (seat/ACL denied in sandboxes) reported gracefully, not a crash.

   **Acceptance Criteria**
   a. [ ] FPS mouse-look works via pointer-lock on Wayland; no assumption of warp/positioning.
