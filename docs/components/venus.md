# venus — Vulkan-over-virtio

`github.com/go-virtio/venus` is the pure-Go (CGO=0) implementation of
**Venus**, the Vulkan-over-virtio protocol — the guest-side counterpart
to the [`gpu`](gpu.md) virtio-gpu work. Venus reuses the virtio-gpu
*device* but replaces the whole submission model and serialises
essentially the entire Vulkan API, so it is a genuinely larger
undertaking than the virgl/GL path. This repo de-risks it bottom-up, one
verifiable rung at a time.

## What it provides

- **`internal/vncs`** — the wire runtime (the `vn_cs.h` equivalent): an
  `Encoder` and a mirroring `Decoder` over a `[]byte`, with LE,
  4-byte-aligned primitives (`Uint32/Int32/Uint64/Float32/Flags/Bool32/
  DeviceSize/ArraySize/SimplePointer/BlobArray/String/Handle/Result`)
  plus fixed/union array helpers. Each method transcribes a specific Mesa
  `vn_encode_*` / `vn_decode_*` function, cited inline.
- **`gen`** — parses `vk.xml` (`encoding/xml`) into a typed model and
  emits Go encoders **and decoders** by walking struct/command members
  exactly as Mesa's Python generator does (honouring `optional` / `len` /
  `altlen` / `sType` / `pNext`, enums, handles, `VkBool32` /
  `VkDeviceSize` / `size_t`, nested-by-value structs, fixed-size arrays,
  the `VkClearColorValue` union, counted arrays, typed `pNext` extension
  chains, count+array reply decoders, and returned-only struct decoders).
- **`cmd/vkgen`** — the generator CLI (`-xml`, `-out`, `-pkg`).
- **`proof`** — a generated proof subset spanning the **full clear-image
  → readback** command closure (`vkCreateInstance/Device/Image`,
  `vkAllocateMemory`, `vkCmdClearColorImage`, `vkQueueSubmit`, …) whose
  encoded/decoded bytes are asserted against independently hand-derived
  Mesa bytes — no GPU, no host required.

## End-to-end

Beyond the offline-verified generator, Venus carries a working
shared-memory ring transport. A clear-image runs end-to-end on a real
renderer: the guest submits the full Vulkan command sequence
(instance → device → image → clear → submit) over the ring to
`virgl_test_server --venus` + lavapipe, and the host creates the image
and executes the clear. **Guest-side pixel readback is closed too**: on a
Linux render-node host the guest maps the cleared image's
`HOST3D|MAPPABLE` blob (gbm dma_buf) and reads the texels back as RED.

## License

BSD-3-Clause.
