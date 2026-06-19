# vsock — virtio-vsock

`github.com/go-virtio/vsock` is a pure-Go virtio-vsock (socket device)
driver for the standard PCI-bound device (VID 0x1AF4, DID 0x1053). It
implements the modern-transport (Virtio 1.0+) init sequence and the
three-virtqueue packet path.

## Scope

It owns device bring-up, the three virtqueues (rx / tx / event, Virtio
1.1 §5.10.2), and the on-the-wire `struct virtio_vsock_hdr` marshalling,
exposing a **packet-level** `SendPacket` / `ReceivePacket` API.

It deliberately does **not** implement the connection state machine or
the credit-based flow control (`buf_alloc` / `fwd_cnt` accounting) —
those belong a layer up, exactly as [`net`](net.md) drives frames rather
than TCP. The header's addressing and credit fields are surfaced on
`Packet` so the upper layer can implement them.

## Quick start

```go
import virtiovsock "github.com/go-virtio/vsock"

vs, err := virtiovsock.OpenVirtioVsock(transport)
if err != nil {
    return err
}

// Connection-request packet to the host (CID 2), port 5000.
err = vs.SendPacket(virtiovsock.Packet{
    SrcCID:  vs.GuestCID,
    DstCID:  virtiovsock.CIDHost,
    SrcPort: 1024,
    DstPort: 5000,
    Type:    virtiovsock.TypeStream,
    Op:      virtiovsock.OpRequest,
})

pkt, err := vs.ReceivePacket(10000) // busy-poll budget
```

`OpenVirtioVsock` leaves the device in DRIVER_OK state with the rx and
event queues pre-posted and `GuestCID` populated from device config.

## License

BSD-3-Clause.
