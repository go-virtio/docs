# gpu — virtio-gpu (2D + virgl 3D + soft3d)

`github.com/go-virtio/gpu` is a pure-Go virtio-gpu driver for the
standard PCI-bound device (VID 0x1AF4, DID 0x1050). It implements the
modern-transport (Virtio 1.0+) init sequence and the control-queue
command path.

Like the sibling drivers it owns device bring-up, both virtqueues — the
**control queue** (controlq, carrying every command) and the **cursor
queue** (cursorq) — and the on-the-wire virtio-gpu control protocol.
Every command is a 2-descriptor chain (a read-only request followed by a
device-writable response), built with `common.AddChain`.

## 2D framebuffer

The base path is a scanout-enumeration + framebuffer API:

- `DisplayInfo` lists the device's scanouts (`GET_DISPLAY_INFO`).
- `SetupFramebuffer` creates a host 2D resource, attaches guest backing,
  and binds it to a scanout (`RESOURCE_CREATE_2D` +
  `RESOURCE_ATTACH_BACKING` + `SET_SCANOUT`). The returned
  `Framebuffer.Pix` is a BGRA byte buffer the caller draws into.
- `Framebuffer.Flush` pushes the drawn pixels to the host and refreshes
  the scanout (`TRANSFER_TO_HOST_2D` + `RESOURCE_FLUSH`).

```go
import virtiogpu "github.com/go-virtio/gpu"

g, err := virtiogpu.OpenVirtioGPU(transport)
displays, err := g.DisplayInfo()
d := displays[0] // first scanout

fb, err := g.SetupFramebuffer(d.ScanoutID, d.Width, d.Height)
// Draw BGRA pixels into fb.Pix, then push to the display.
err = fb.Flush()
```

## The 3D split

"Pure-Go 3D" is three different things, and this module makes the split
explicit:

- **Software (CPU).** `gpu/soft3d` is a dependency-free, z-buffered
  triangle rasterizer that renders into the virtio-gpu framebuffer. Works
  on any host, no GPU.
- **virgl (host GPU).** `gpu` hand-encodes the virgl command stream
  (shaders shipped as TGSI text) so a host `virglrenderer` does the
  drawing — real hardware acceleration, still CGO=0. `ClearScreen`,
  `DrawTriangle` and `DrawTexturedTriangle` are all validated against a
  real virglrenderer (software llvmpipe) via the
  [`validate`](validate.md) harness.
- **Vulkan / Venus.** handled by the separate [`venus`](venus.md) module.

## License

BSD-3-Clause.
