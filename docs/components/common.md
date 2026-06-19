# common — transport-agnostic infrastructure

`github.com/go-virtio/common` hosts the shared building blocks that every
virtio device-class driver needs and that do not themselves depend on a
particular host transport (UEFI, bare-metal MMIO, virtio-mmio,
vhost-user, …). It mirrors the Linux kernel's `<linux/virtio.h>`
shared-infrastructure pattern: per-class drivers import this package for
the transport-independent pieces and provide their own spec-level driver
on top.

## What it provides

- **PCI capability walker** (`pci.go`) — parses the standard
  `struct virtio_pci_cap` chain published by every modern virtio device
  (Virtio 1.1 §4.1.4). Driven through a `PCIConfigReader` interface so
  the same walker covers any host.
- **Modern transport layout** (`modern.go`) — the `ModernConfig` handle
  that pins the four required + one optional PCI capabilities
  (COMMON_CFG / NOTIFY_CFG / ISR_CFG / DEVICE_CFG / PCI_CFG) plus the
  typed register accessors that route through a `BARMemoryAccessor`.
  Covers the full Virtio 1.1 §4.1.5 register table.
- **Split-virtqueue layout + driver-side state machine**
  (`virtqueue.go`) — descriptor table, available ring, used ring, plus
  the `AddBuffer` / `AddChain` / `PostAvail` / `PollUsed` / `Reclaim`
  bookkeeping. Backing pages come from a `PageAllocator`.
- **Device-class IDs** — the centralized virtio device IDs (including the
  virtio-fs IDs).
- **Transport interfaces** (`transport.go`) — `PCIConfigReader`,
  `BARMemoryAccessor`, `PageAllocator`, `Transport`.

!!! note "Endianness"
    virtio rings and config registers are little-endian on the wire. The
    `common` ring/register code carries the byte-order handling for the
    whole driver set; its ring code was recently corrected for
    big-endian hosts.

## License

BSD-3-Clause.
