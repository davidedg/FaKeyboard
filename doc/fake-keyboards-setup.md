# Emulating two keyboards for ckb-next daemon development (profiles/modes)

A concrete setup on top of FaKeyboard (see the repo [README](../README.md) for what this tool is
in general): two emulated Corsair keyboards, running without any real hardware, for developing and
testing changes to ckb-next's on-device profile/mode handling (`profile.c`, `profile_keyboard.c`,
`command.c`) — plus simulating physical keypresses to test key-binding (`bind`/`unbind`/`rebind`).

FaKeyboard only emulates HID descriptors and the handful of vendor reports coded into each
`hid-*.c` — it does **not** implement every Corsair vendor-specific feature report. In particular,
reading back an on-device hardware profile isn't implemented, so the ckb-next daemon logs
`Bad input header` / `nxp_usb_read: Timeout` / `Unable to load hardware profile` warnings during
connect and falls back to a blank/default profile. Setup still completes (`cmd`/`notify0`/`notify1`
FIFOs come up), which is enough to exercise profile/mode *command handling* (writes to `cmd`,
events on `notify*`) — just don't expect faithful persistence/read-back of profile state.

## Design notes

- `usbip.c`'s `usbip_run()` hardcoded the USB/IP TCP port to `3240` (`TCP_SERV_PORT`), which meant
  only one `hid-*` binary could listen at a time on one host. Reads the port from the
  `FAKEYBOARD_PORT` env var instead (falls back to 3240 if unset). Run each device on its own port
  and select it client-side with `usbip --tcp-port <port> ...`.
- `Makefile` originally built with `-fsanitize=address`. ASan requires being the first thing
  preloaded into the process, which conflicts with running the binary under `stdbuf` (used by
  `fake-keyboard` to force line-buffered log output — otherwise the "Listening on TCP port" line
  sits in a full stdio buffer and never reaches the log file). Dropped `-fsanitize=address` from
  `CFLAGS` since these run as long-lived background test servers, not something being
  memory-debugged.
- `usbip.c`'s `usbip_run()` blocked on `recv()` alone. Replaced with `poll()` over
  `[sockfd, ctrlfd]`, where `ctrlfd` is an optional control FIFO (path from the
  `FAKEYBOARD_CTRL_FIFO` env var) — only created if a `hid-*.c` program sets the new
  `ctrl_line_handler` function pointer before calling `usbip_run()` (default `NULL`, so every
  binary except `hid-keyboard-special.c` is unaffected). Added `send_async_response()`, a
  single-`send()` variant of `send_corsair_response()` for completing a stashed URB from outside
  the normal request/response flow (used to inject synthetic keypresses — see below). Single
  thread throughout, no `-lpthread` — see "Simulating a physical keypress" for why this was safe.
- `hid-keyboard-special.c` (K68 only) gained a `k68_handle_ctrl_line()` handler wired to the
  control FIFO above, plus `devid81`/`pending_release` state extending the pre-existing `ep==0x01`
  branch — see "Simulating a physical keypress" below for the full mechanism.

## The two devices in use

Picked as the simplest, most contrasting pair of *wired* keyboards using the standard on-device
(non-file-based) profile storage — i.e. neither `USES_FILE_HWSAVE`, wireless, nor Bragi, so the
comparison isolates the plain NXP on-device profile/mode path from its "override" variant:

| Binary | Real device | PID | `src/daemon/usb.h` flags |
|---|---|---|---|
| `hid-keyboard` | Corsair Strafe (non-RGB) | `1b1c:1b15` (`P_STRAFE_NRGB`) | plain NXP path, no override |
| `hid-keyboard-special` | Corsair K68 | `1b1c:1b4f` (`P_K68`) | `IS_V2_OVERRIDE`/`IS_V3_OVERRIDE` |

Build both with `make hid-keyboard hid-keyboard-special` from `src/` (see the repo README — the
Makefile defines about a dozen more targets for other devices, not needed here).

## Simulating a physical keypress (K68 only)

`fake-keyboard key k68 <name>` makes the daemon see a real keydown+keyup for the named key on K68
— the actual USB path a real keypress takes, not a shortcut into the daemon. Use this to test
`bind`/`unbind`/`rebind` end-to-end. Not implemented for Strafe (see "Design notes" above) or for
any endpoint other than K68's EP 0x81.

**Why EP 0x81, not EP 0x82**: the obvious guess (based on the config descriptor declaring EP
0x82 as a second "software mode" endpoint) is wrong for this specific fake device. `hid-keyboard-special.c`'s
descriptor declares only **2 USB interfaces**, so the daemon's `kb->epcount == 2` and
`nxp_fill_input_eps()` (`src/daemon/usb_nxp.c:5`) ends up polling **only EP 0x81** — EP 0x82 is
declared but never read by the daemon for this device. The emulator's own command-response code
already reused EP 0x81 (via the stashed `seqnum81`) for the identification-packet reply before this
change; the keypress injection follows the same, already-working, channel. The daemon's routing is
content-based, not endpoint-based (`process_input_urb`, `src/daemon/keymap.c:810` /`:940-965`): a
64-byte packet starting with `0x03` (`CORSAIR_IN`) is parsed as a keypress via `corsair_kbcopy()`
regardless of which endpoint it arrived on, so reusing EP 0x81 for this is legitimate, not a hack.

Packet format sent on EP 0x81 (`send_async_response()` in `usbip.c`, called from
`k68_handle_ctrl_line()` in `hid-keyboard-special.c`):
```
byte 0:     0x03 (CORSAIR_IN)
byte 1-19:  bitmap, bit i = keymap[i] pressed (i = 0-based position in src/daemon/keymap.c's
            keymap[] array, NOT its "scan" hex field)
byte 20-63: padding
```
`corsair_kbcopy()` is a flat `memcpy` (whole-state, not a delta), so a keydown must always be
followed by an all-zero keyup or the key stays "held" forever — `hid-keyboard-special.c` does this
automatically: sending `key <idx>` completes the currently-stashed EP 0x81 URB with the keydown
report and arms `pending_release`, which the *next* EP 0x81 poll (the kernel resubmits within
~1ms) auto-completes with the keyup.

The device must be `active` first (`fake-keyboard cmd k68 active`) — `bind`/`rebind` only take
effect when `kb->active` (`src/daemon/input.c:347`). This already works unmodified: `setactive_kb`
(`src/daemon/device_keyboard.c:22`) only writes commands, never waits for a specific reply, and
the emulator's EP2-OUT command channel acks every write unconditionally.

**`unbind` vs `rebind`** (learned the hard way writing the verification test): `unbind <key>` sets
the binding to `KEY_UNBOUND` (`-3`), which *suppresses* the key entirely (`src/daemon/input.c:396`)
— it does not restore the physical key. For a negative control ("does the *unmodified* key still
come through"), use `rebind <key>` instead, which restores `keymap[keyindex].scan`
(`src/daemon/input.c:622-630`).

Verified end-to-end: `active` → `bind a:b` → `key a` → uinput emits keycode 48 (`b`) down/up →
`rebind a` → `key a` → uinput emits keycode 30 (`a`) down/up. Read the daemon's uinput node via
`/proc/bus/input/devices` (`Name="ckb2: ..."`, format `"ckb%d: %s"` from
`src/daemon/input_linux.c:96`) to find the right `/dev/input/eventN` — reading it requires root or
membership in the `input` group.

## Control script

One-time system deps (Arch): `sudo pacman -S usbip` (client + protocol lib) and
`sudo modprobe vhci-hcd` (kernel module, present in-tree, just needs loading). Then, from the repo
root:

```bash
src/fake-keyboard start            # both devices: build if needed, launch, attach
src/fake-keyboard status           # server / usbip attach / daemon-node state
src/fake-keyboard cmd strafe mode 1   # write a raw ckb-next command to ckb1's cmd FIFO
src/fake-keyboard key k68 a        # simulate a physical "a" keypress+release (k68 only)
src/fake-keyboard logs k68         # tail the USB/IP traffic log for one device
src/fake-keyboard stop             # detach + kill both
```

Run `src/fake-keyboard` with no args for full usage. It tracks servers it started itself (PID/log
files under `$XDG_RUNTIME_DIR/fake-keyboard-$USER`); it won't manage a `hid-keyboard*` process
started by hand outside of it.

Manual equivalent, if working outside the script:

```bash
cd src
FAKEYBOARD_PORT=3240 ./hid-keyboard &                        # Strafe
FAKEYBOARD_PORT=3241 ./hid-keyboard-special &                # K68
sudo usbip attach -r 127.0.0.1 -b 1-1                        # Strafe -> /dev/input/ckb1
sudo usbip --tcp-port 3241 attach -r 127.0.0.1 -b 1-1         # K68    -> /dev/input/ckb2
sudo usbip port                                               # lists attached vhci ports
```

To detach: `sudo usbip detach -p <port>` (port numbers from `usbip port`, e.g. `00`/`01`), then
kill the two `hid-keyboard*` processes (or just use `fake-keyboard stop`).

## Gotchas hit during setup

- Running `usbip attach` twice against the *same* port/busid (e.g. forgetting `--tcp-port` for the
  second device) doesn't error cleanly — it hangs the client process waiting on the fake server,
  which is already mid-connection with the first client. Kill the hung `usbip`/`sudo` processes and
  retry with the correct `--tcp-port`.
- Attaching a second device while the daemon is still mid-handshake with the first one (it retries
  `nxp_usb_read` for ~10s before giving up and falling back to a blank profile — see above) makes
  `usbip attach` intermittently fail with `open vhci_driver (is vhci_hcd loaded?)`. Harmless and
  self-clears; `fake-keyboard` retries the attach (up to 10x, 2s apart) to ride it out.
