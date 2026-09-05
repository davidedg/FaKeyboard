# FaKeyboard

FaKeyboard is a USB device emulator for [ckb-next](https://github.com/ckb-next/ckb-next), the
Linux driver for Corsair keyboards and mice. It presents fake Corsair USB devices to the Linux
kernel over USB/IP, so ckb-next's daemon can be developed and tested without any real hardware.

Each `src/hid-*.c` program is a small USB/IP server emulating one Corsair device (keyboard, mouse,
mousepad, headset, and a few others). The Linux `vhci-hcd` kernel driver attaches to it over the
network, and the kernel then sees a genuine USB device — indistinguishable from real hardware to
udev, ckb-next, or any other USB software.

Based on the USB/IP hardware emulation library by Luis Claudio Gambôa Lopes.

## Building

From `src/`:

```
make <target>
```

`<target>` is one of the programs listed in `src/Makefile`'s `PROGS` (for example `hid-keyboard`,
`hid-keyboard-special`, `hid-mouse`). Running `make` with no target builds all of them.

## Setup and usage

- [doc/fake-keyboards-setup.md](doc/fake-keyboards-setup.md) — setting up the environment (USB/IP
  kernel module, `usbip` client) and connecting an emulated device to ckb-next.
- [src/fake-keyboard](src/fake-keyboard) — a control script to start/stop the emulated devices,
  send commands to ckb-next through them, and simulate key presses.

## Protocol reference

[doc/usbip_protocol.txt](doc/usbip_protocol.txt) documents the USB/IP protocol subset this tool
implements.

## License

GPL-2.0, see [LICENSE](LICENSE).
