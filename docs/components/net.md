# net — virtio-net

`github.com/go-virtio/net` is a pure-Go virtio-net driver for the
standard PCI-bound virtio-net device (VID 0x1AF4, DID 0x1041). It
implements the modern-transport (Virtio 1.0+) init sequence and the
split-virtqueue TX/RX path.

This package owns the spec-level driver — the per-frame header layout,
the feature-acceptance mask, the init sequence (Virtio 1.1 §3.1.1), and
the rxq/txq state machine — and routes every transport-level operation
through [`common`](common.md)'s `Transport` interface. It is the
reference per-device-class driver the other drivers mirror.

## Quick start

```go
import virtionet "github.com/go-virtio/net"

// transport is any value that implements go-virtio/common.Transport.
vn, err := virtionet.OpenVirtioNet(transport)
if err != nil {
    return err
}
if err := vn.TransmitFrame(ethFrame); err != nil {
    return err
}
frame, err := vn.ReceiveFrame(10000) // poll budget
```

## License

BSD-3-Clause.
