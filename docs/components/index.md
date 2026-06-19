# Components

`go-virtio` is a set of dependency-free Go modules (standard library
only, `CGO_ENABLED=0`) layered around one shared infrastructure package
and one narrow `Transport` interface. Each driver owns its spec-level
device class and consumes `common` for everything transport-related.

## Core

| Module | Import path | What it does |
|--------|-------------|--------------|
| [`common`](common.md) | `github.com/go-virtio/common` | Transport-agnostic infrastructure: PCI capability walker, modern-config register layout, split-virtqueue + descriptor chaining, device-class IDs, and the `Transport` / `PCIConfigReader` / `BARMemoryAccessor` / `PageAllocator` interfaces. |

## Drivers

| Module | Import path | Device (DID) |
|--------|-------------|--------------|
| [`net`](net.md) | `github.com/go-virtio/net` | virtio-net (0x1041) |
| [`rng`](rng.md) | `github.com/go-virtio/rng` | virtio-rng (0x1044) |
| [`vsock`](vsock.md) | `github.com/go-virtio/vsock` | virtio-vsock (0x1053) |
| [`blk`](blk.md) | `github.com/go-virtio/blk` | virtio-blk (0x1042) |
| [`console`](console.md) | `github.com/go-virtio/console` | virtio-console (0x1043) |
| [`balloon`](balloon.md) | `github.com/go-virtio/balloon` | virtio-balloon (0x1045) |
| [`fs`](fs.md) | `github.com/go-virtio/fs` | virtio-fs (0x105A) |
| [`input`](input.md) | `github.com/go-virtio/input` | virtio-input (0x1052) |
| [`sound`](sound.md) | `github.com/go-virtio/sound` | virtio-sound (0x1059) |

## GPU

| Module | Import path | What it does |
|--------|-------------|--------------|
| [`gpu`](gpu.md) | `github.com/go-virtio/gpu` | virtio-gpu (0x1050): 2D framebuffer + virgl 3D + the `gpu/soft3d` software rasterizer. |
| [`venus`](venus.md) | `github.com/go-virtio/venus` | Vulkan-over-virtio (Venus): `vk.xml`→Go serializer/generator + ring transport. |

## Validation

| Module | Import path | What it does |
|--------|-------------|--------------|
| [`validate`](validate.md) | `github.com/go-virtio/validate` | Real-hardware (TamaGo + QEMU) validation harness and a pure-Go virglrenderer/Venus vtest client. |
