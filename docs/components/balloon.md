# balloon — virtio-balloon

`github.com/go-virtio/balloon` is a pure-Go virtio-balloon (memory
balloon) driver for the standard PCI-bound device (VID 0x1AF4, DID
0x1045). It implements the modern-transport (Virtio 1.0+) init sequence
and the two-virtqueue page-transfer path.

virtio-balloon (Virtio 1.1 §5.5) lets the host reclaim guest RAM on
demand. "Inflating" the balloon hands guest pages back to the host
(shrinking the guest's effective memory); "deflating" reclaims them. The
device-config `num_pages` field is the host's desired balloon size in
4096-byte pages.

Only `VIRTIO_F_VERSION_1` is negotiated — in particular
`VIRTIO_BALLOON_F_STATS_VQ` is **not** negotiated, so the device exposes
exactly two queues (inflateq = 0, deflateq = 1) and there is no stats
queue.

## Quick start

```go
import virtioballoon "github.com/go-virtio/balloon"

vb, err := virtioballoon.OpenVirtioBalloon(transport)
if err != nil {
    return err
}

// vb.NumPages is the host's desired balloon size (4096-byte pages),
// read from device config. Inflate toward it, deflate away from it.
if err := vb.Inflate(64); err != nil { // hand 64 pages to the host
    return err
}
if err := vb.Deflate(32); err != nil { // reclaim 32 of them
    return err
}
// vb.Actual tracks the driver-side current balloon size.
```

A request packs an array of le32 page-frame-numbers (`phys >> 12`) into a
single device-readable DMA buffer (at most 256 PFNs per buffer; larger
requests are chunked), posts it to inflateq or deflateq, rings the
doorbell, and busy-polls the used ring.

!!! note "`actual` is tracked driver-side only"
    The spec asks the driver to write the current balloon size back to
    the device-config `actual` field (Virtio 1.1 §5.5.6.1). The driver
    tracks `Actual` on the driver side rather than writing it back to the
    device config.

## License

BSD-3-Clause.
