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
