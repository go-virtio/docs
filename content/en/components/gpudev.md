---
title: "gpudev — the virtio-gpu host (device model + WebGPU backend)"
weight: 165
---

# gpudev — the virtio-gpu host {#gpudev}

`github.com/go-virtio/gpudev` is the **host side** of virtio-gpu. Every
other go-virtio module is a guest driver; gpudev is the pure-Go device
that guest drivers talk to. go-virtio/gpu, the Linux `virtio_gpu`
driver, or any other driver can use it.

| package | what |
|---|---|
| `gpudev` | the device model: 2D resources, backing, `TRANSFER_TO_HOST_2D`, scanouts, flushes, the capset queries, and webgpu-batch contexts |
| `gpudev/virtq` | the device side of a split virtqueue, which checks everything in the rings |
| `gpudev/pcisim` | an in-process virtio-pci modern transport that implements `go-virtio/common.Transport` |
| `gpudev/memback` | an in-memory backend for tests and fuzzing |
| `gpudev/webgpuback` | the browser backend on [go-gfx/webgpu](https://github.com/go-gfx/webgpu) (GOOS=js) |

```go
ram := &gpudev.RAM{Base: 1 << 30, Bytes: make([]byte, 64<<20)}
dev := gpudev.New(gpudev.Config{Scanouts: []gpudev.Rect{{Width: 1024, Height: 768}}}, ram, backend)
tr := pcisim.New(dev, ram)

g, _ := gpu.OpenVirtioGPU(tr) // the go-virtio/gpu guest driver, unchanged
fb, _ := g.SetupFramebuffer(0, 1024, 768)
fb.Flush()
```

## WebGPU backend {#webgpu-backend}

- **2D.** Each resource is a GPU texture. `TRANSFER_TO_HOST_2D` is one
  `writeTexture`, taken straight from guest RAM when the rows are
  contiguous. `RESOURCE_FLUSH` is one GPU `copyTextureToTexture` into the
  canvas texture, followed by present.
- **webgpu-batch.** Objects are created from go-gfx/webgpu wire
  descriptors, and streams are submitted with `Device.Replay`. Guest
  object ids are private to their context and are translated before the
  backend sees them. A stream can only name objects its own context
  holds.

## Security {#security}

The guest is untrusted:

- every id, size, offset, count and rectangle is bounded in 64-bit
  arithmetic;
- quotas bound host memory, resources, backing entries and contexts;
- rings are fully checked;
- requests are copied out of guest memory before they are decoded;
- a webgpu-batch frame that fails loses its context.

`FuzzHandleControl` and `FuzzPop` run in CI.

## Status {#status}

- **2D and webgpu-batch are validated in a browser on a real GPU**, with
  the go-virtio/gpu driver unchanged. The canvas texture equals the
  guest's framebuffer, and a guest compute pipeline doubles a buffer on
  the GPU. This ran on Apple Metal locally and on macOS CI with a GPU
  required.
- Not yet run under a hypervisor or with the Linux driver.
- Not implemented yet:
  - `BIND_RESOURCE` readback on the WebGPU backend;
  - the A8/X8-first formats.

## License {#license}

BSD-3-Clause.
