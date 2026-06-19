# go-virtio

**Pure-Go, transport-agnostic [virtio](https://docs.oasis-open.org/virtio/virtio/v1.1/virtio-v1.1.html)
guest drivers** — no cgo, no kernel.

`go-virtio` is a family of Go modules that implement virtio device-class
drivers in pure Go and route every transport-level operation through one
narrow interface, so the same driver code works under UEFI's
`EFI_PCI_IO_PROTOCOL`, bare-metal MMIO, virtio-mmio, TamaGo, and anything
else that can satisfy the contract. It replaces the fragmented in-tree
virtio code that each Go project reinvents with one reusable,
well-tested, transport-pluggable driver set. Extracted from the
[cloud-boot](https://github.com/cloud-boot) TamaGo + UEFI loader, but
designed to run anywhere virtio runs.

## How the pieces fit

```
  net  rng  vsock  blk  console  balloon  fs  input  sound  gpu  ← spec-level drivers
   └────┴─────┴─────┴──────┴────────┴──────┴─────┴──────┴─────┘
                            │
              ┌─────────────▼──────────────┐
              │       go-virtio/common      │  transport-agnostic infra
              │  PCI cap walker · ModernConfig · split-virtqueue ·
              │  descriptor chaining · device IDs · Transport interface
              └─────────────┬──────────────┘
                            │  Transport (PCIConfigReader,
                            │   BARMemoryAccessor, PageAllocator)
   ┌────────────────────────▼──────────────────────────────────────┐
   │  EFI_PCI_IO_PROTOCOL · bare-metal MMIO · virtio-mmio · TamaGo … │  host backplane
   └────────────────────────────────────────────────────────────────┘
```

Every driver consumes [`common.Transport`](components/common.md) and
nothing else. Swap the backplane, keep the driver.

!!! note "virtio is little-endian"
    The virtio rings and config registers are defined little-endian on
    the wire. The shared `common` virtqueue/register code carries the
    byte-order handling for the driver set (its ring code was recently
    fixed for big-endian hosts).

## The drivers

| Module | Device (DID) | What it does |
|--------|--------------|--------------|
| [`common`](components/common.md) | — | Shared infrastructure: PCI cap walker, modern-config registers, split-virtqueue + descriptor chaining, device-class IDs, the `Transport` interface. |
| [`net`](components/net.md) | virtio-net (0x1041) | Frame-level `TransmitFrame` / `ReceiveFrame` over a TX/RX queue pair. |
| [`rng`](components/rng.md) | virtio-rng (0x1044) | Single-queue entropy `Read` — the minimal device class. |
| [`vsock`](components/vsock.md) | virtio-vsock (0x1053) | Three queues; packet-level `SendPacket` / `ReceivePacket` with `virtio_vsock_hdr`. |
| [`blk`](components/blk.md) | virtio-blk (0x1042) | `ReadBlocks` / `WriteBlocks` / `Flush`; header + data + status descriptor chains. |
| [`console`](components/console.md) | virtio-console (0x1043) | Raw byte-stream `Write` / `Read` over an rx/tx pair. |
| [`balloon`](components/balloon.md) | virtio-balloon (0x1045) | `Inflate` / `Deflate` via le32 page-frame-number arrays. |
| [`fs`](components/fs.md) | virtio-fs (0x105A) | FUSE-over-virtio **read-write** mount: Lookup/Open/Read + Write/Create/Mkdir/SetAttr/Rename/… |
| [`input`](components/input.md) | virtio-input (0x1052) | Keyboard + relative-pointer event read path (`input_event` wire format). |
| [`sound`](components/sound.md) | virtio-sound (0x1059) | Minimal PCM playback + capture over the control / tx / rx queues. |
| [`gpu`](components/gpu.md) | virtio-gpu (0x1050) | **2D framebuffer** + **virgl 3D** + a pure-Go **software 3D rasterizer** (`gpu/soft3d`). |
| [`venus`](components/venus.md) | virtio-gpu | **Vulkan-over-virtio** (Venus): a `vk.xml`→Go serializer/generator plus a ring transport; clear-image end-to-end with guest-side pixel readback. |
| [`validate`](components/validate.md) | — | Real-hardware validation harness (TamaGo + QEMU) and a pure-Go virglrenderer/Venus vtest client. |

## The 3D story

`go-virtio/gpu` makes the honest split explicit — "pure-Go 3D" is three
different things:

- **Software (CPU).** `gpu/soft3d` is a dependency-free, z-buffered
  triangle rasterizer that renders into the virtio-gpu framebuffer. Works
  on any host, no GPU.
- **virgl (host GPU).** `gpu` hand-encodes the virgl command stream
  (shaders shipped as TGSI text) so a host `virglrenderer` does the
  drawing — real hardware acceleration, still CGO=0.
- **Vulkan / Venus.** [`venus`](components/venus.md) serialises the Vulkan
  API over a virtio ring; a clear-image runs end-to-end on a real
  renderer, with guest-side pixel readback on a Linux render-node host.

See [Components](components/index.md) for the per-module pages.
