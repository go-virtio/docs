# rng — virtio-rng

`github.com/go-virtio/rng` is a pure-Go virtio-rng (virtio-entropy)
driver for the standard PCI-bound device (VID 0x1AF4, DID 0x1044). It
implements the modern-transport (Virtio 1.0+) init sequence and the
single-virtqueue entropy-read path.

virtio-rng is the simplest device class in the spec (Virtio 1.1 §5.4):
one virtqueue (the "requestq"), no device-specific feature bits, and no
device-config region. The driver posts a device-writable buffer; the
device fills it with random bytes and reports — via the used ring — how
many bytes it wrote. Every transport-level operation routes through
[`common`](common.md)'s `Transport` interface.

## Quick start

```go
import virtiorng "github.com/go-virtio/rng"

vr, err := virtiorng.OpenVirtioRng(transport)
if err != nil {
    return err
}

// Read always fills the whole buffer on success (io.ReadFull-style,
// matching crypto/rand's Reader contract).
buf := make([]byte, 32)
if _, err := vr.Read(buf); err != nil {
    return err
}

// ReadPoll takes an explicit busy-poll budget for tighter timeouts.
n, err := vr.ReadPoll(buf, 50000)
```

`OpenVirtioRng` leaves the device in DRIVER_OK state with an empty,
ready request queue; the driver posts a buffer on demand in `Read`.

## License

BSD-3-Clause.
