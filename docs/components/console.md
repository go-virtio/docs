# console — virtio-console

`github.com/go-virtio/console` is a pure-Go virtio-console driver for the
standard PCI-bound device (VID 0x1AF4, DID 0x1043). It implements the
modern-transport (Virtio 1.0+) init sequence and the raw byte-stream
RX / TX path.

This package targets the single-port baseline (Virtio 1.1 §5.3): it
negotiates only `VIRTIO_F_VERSION_1`, so `VIRTIO_CONSOLE_F_MULTIPORT` is
not acknowledged and the device exposes exactly two virtqueues — a
receiveq (port 0 input) and a transmitq (port 0 output). There are no
control queues and no per-message header: the console is a raw
bidirectional byte stream, simpler than virtio-net (no `virtio_net_hdr`
to prepend or strip). `VIRTIO_CONSOLE_F_SIZE` is masked out, so the
driver performs no device-config reads.

The driver pre-posts one-page device-writable buffers on the receiveq at
bring-up so the device has somewhere to land guest input, and posts
device-readable buffers on demand in `Write`.

## Quick start

```go
import virtioconsole "github.com/go-virtio/console"

vc, err := virtioconsole.OpenVirtioConsole(transport)
if err != nil {
    return err
}

// Write sends raw bytes to the console output, chunked one page at a time.
if _, err := vc.Write([]byte("hello from the guest\n")); err != nil {
    return err
}

// Read polls the receiveq for one buffer of console input. The argument
// is a busy-poll budget; ErrReceiveTimeout is returned if exhausted.
in, err := vc.Read(10000)
```

## License

BSD-3-Clause.
