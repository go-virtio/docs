# input — virtio-input

`github.com/go-virtio/input` is a pure-Go virtio-input driver for the
standard PCI-bound device (VID 0x1AF4, DID 0x1052). It implements the
modern-transport (Virtio 1.0+) init sequence and the device-to-guest
event read path.

This package targets the keyboard + relative-pointer baseline (Virtio
1.2 §5.8): it negotiates only `VIRTIO_F_VERSION_1` and the device
exposes two virtqueues — `eventq` (queue 0, device-to-guest events) and
`statusq` (queue 1, guest-to-device status such as LED + force-feedback,
not driven by this MVP). The event wire format mirrors Linux's
`input_event` structure (type, code, value), and the key-code constants
are a hand-picked subset of Linux's `input-event-codes.h` — enough for
keyboard input plus relative mouse events (the DOOM port's input budget).

The driver pre-posts buffers on the eventq at bring-up and re-posts the
same buffer after each successful `ReadEvent`.

## Quick start

```go
import virtioinput "github.com/go-virtio/input"

vi, err := virtioinput.OpenVirtioInput(transport)
if err != nil {
    return err
}

// Blocking read (busy-polls until an event is consumed).
ev, err := vi.ReadEvent(true)
if err != nil {
    return err
}
switch ev.Type {
case virtioinput.EvKey:
    // ev.Code is e.g. KeyA, KeySpace, BtnLeft; ev.Value is 1 (down),
    // 0 (up), or 2 (auto-repeat).
case virtioinput.EvRel:
    // ev.Code is RelX / RelY / RelWheel; ev.Value is the signed delta.
case virtioinput.EvSyn:
    // End-of-event-group marker (SYN_REPORT / SYN_DROPPED / ...).
}

// Non-blocking read (returns ErrEventNotReady if the eventq is empty).
ev, err = vi.ReadEvent(false)
```

## License

BSD-3-Clause.
