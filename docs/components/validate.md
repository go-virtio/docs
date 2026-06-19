# validate — real-hardware validation harness

`github.com/go-virtio/validate` is a **real-hardware validation harness**
for the go-virtio guest drivers. It boots a bare-metal
[TamaGo](https://github.com/usbarmory/tamago) guest under QEMU, drives
real virtio devices with the pure-Go go-virtio drivers, and asserts the
result — going beyond unit tests with a fake device.

## What it proves (end-to-end)

A TamaGo guest enumerates PCI, binds a `common.Transport` to a QEMU
device, and runs the real driver path. For [`gpu`](gpu.md):
`OpenVirtioGPU → DisplayInfo → SetupFramebuffer → soft3d.RenderCube →
Flush`, and the host `screendump` shows a shaded 3D cube:

```
VALIDATE: GPU=0x1af4:0x1050 scanouts=1
VALIDATE: fb 320x240 resource=1 pixbytes=307200
VALIDATE: checksum=0xdcd3247a nonzero_pixels=18168 distinct_colors=5
RESULT: PASS (non-uniform frame consistent with a rendered cube)
```

The 2D virtio-gpu path needs no virglrenderer — QEMU's stock device model
serves it — so this validates the 2D driver + the CPU software 3D
rasterizer on a real device model. (This is the weft microVM guest
stack.) Two real bugs surfaced here that a fake-device unit test cannot
catch: the PCI config cap-walk needs dword-granular byte extraction, and
`screendump` must target the virtio-gpu rather than q35's default VGA.

## Arch notes

- **x86_64 under TCG.** TamaGo has no arm64 QEMU-`virt` board and HVF only
  accelerates arm64 guests, so x86_64 TamaGo runs under
  `qemu-system-x86_64` TCG emulation.
- **`board/board.go`** — a local q35/amd64 board that masks the legacy
  8259 PIC (`outb(0x21,0xff)`) after switching IRQ routing to the I/O
  APIC, avoiding a spurious `exception: vector 8` double-fault.
- **`transport.go`** — a `tamagoTransport` implementing `common.Transport`
  with port-mapped PCI config (0xcf8/0xcfc), BAR-window MMIO, and
  `dma.Reserve`-backed page allocation.

## vtest/ — virgl 3D and Venus validation

`vtest/` is a pure-Go (CGO=0) client for virglrenderer's **vtest**
protocol (`virgl_test_server` over a Unix socket — Mesa's CI method,
software-rendered with llvmpipe, no GPU). `vtest/cmd/validator` feeds
[`gpu`](gpu.md)'s actual virgl command buffers to a real
`virgl_test_server`, reads the framebuffer back, and asserts the pixels.
It validates virgl CLEAR / triangle / textured-triangle draws and the
full [`venus`](venus.md) clear-image-with-readback path. The protocol
client is 100%-unit-tested offline.

## License

BSD-3-Clause.
