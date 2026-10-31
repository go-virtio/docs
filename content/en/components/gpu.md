---
title: "gpu — virtio-gpu (2D + virgl 3D + soft3d)"
weight: 160
---

# gpu — virtio-gpu (2D + virgl 3D + soft3d) {#gpu-virtio-gpu-2d-virgl-3d-soft3d}

`github.com/go-virtio/gpu` is a pure-Go virtio-gpu driver for the
standard PCI-bound device (VID 0x1AF4, DID 0x1050). It implements the
modern-transport (Virtio 1.0+) init sequence and the control-queue
command path.

Like the sibling drivers it owns device bring-up, both virtqueues — the
**control queue** (controlq, carrying every command) and the **cursor
queue** (cursorq) — and the on-the-wire virtio-gpu control protocol.
Every command is a 2-descriptor chain (a read-only request followed by a
device-writable response), built with `common.AddChain`.

## 2D framebuffer {#2d-framebuffer}

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

## The 3D split {#the-3d-split}

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
  [`validate`]({{< relref "validate.md" >}}) harness.
- **Vulkan / Venus.** handled by the separate [`venus`]({{< relref "venus.md" >}}) module.

## WebGPU over virtio: `webgpu-batch` (experimental) {#webgpu-batch}

`webgpu-batch` is a private context type that carries
[go-gfx/webgpu](https://github.com/go-gfx/webgpu) Recorder streams from a
guest to a host, which replays them on WebGPU. It is the WebGPU analogue
of Venus. It uses capset **30**, an id the virtio spec has not assigned;
the design doc gives the reasons for that choice.

- `gpu/wgpubatch` holds the capset blob and the frame that rides in
  `SUBMIT_3D`: an encoder and a fuzzed decoder that guest and host share.
- `OpenVirtioGPUWebGPUBatch`, `WebGPUBatchCaps`,
  `CreateWebGPUBatchContext` and `SubmitWebGPUBatch` (with fences and
  ring indices) are the guest calls.

```go
g, err := virtiogpu.OpenVirtioGPUWebGPUBatch(transport)
caps, err := g.WebGPUBatchCaps()          // ErrWebGPUBatchUnavailable if not offered
err = g.CreateWebGPUBatchContext(1, "app")

e := wgpubatch.NewEncoder(caps.MaxFrame)
e.Stream(words, data)                      // one go-gfx/webgpu Recorder submit
frame, err := e.Frame()
err = g.SubmitWebGPUBatch(1, frame, virtiogpu.Fence{On: true, ID: 1})
```

**Status.** The host is [gpudev]({{< relref "gpudev.md" >}}).

- **Validated in process:** the guest calls above run against gpudev
  over an emulated virtio-pci transport.
- **Validated in a browser on a real GPU** (Apple Metal, and macOS CI
  with a GPU required): a compute pipeline built by descriptor doubles a
  buffer.
- **Not yet validated:** running under a hypervisor.

The design doc covers the protocol, the security model and the full
validation table:
[docs/WEBGPU.md](https://github.com/go-virtio/gpu/blob/main/docs/WEBGPU.md).

## License {#license}

BSD-3-Clause.
