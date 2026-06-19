# sound — virtio-sound

`github.com/go-virtio/sound` is a pure-Go virtio-sound driver for the
standard PCI-bound device (VID 0x1AF4, DID 0x1059, device type 25). It
implements the modern-transport (Virtio 1.2+) init sequence and the
minimum PCM playback + capture data paths.

This package targets the single-jack baseline (Virtio 1.2 §5.14): it
negotiates only `VIRTIO_F_VERSION_1`, so `VIRTIO_SND_F_CTLS` and the
control-jack reconfiguration extensions are not acknowledged. The device
exposes four virtqueues — `controlq` (queue 0, control commands),
`eventq` (queue 1, device events, not driven by this MVP), `tx` (queue 2,
PCM playback frames), and `rx` (queue 3, PCM capture frames). The driver
issues the minimal PCM stream lifecycle commands (`PCM_INFO`,
`PCM_SET_PARAMS`, `PCM_PREPARE`, `PCM_START`, `PCM_STOP`, `PCM_RELEASE`)
and routes raw signed 16-bit little-endian (`S16_LE`) frames over the PCM
data queues. Format conversion is the caller's responsibility.

## Quick start

```go
import virtiosound "github.com/go-virtio/sound"

vs, err := virtiosound.OpenVirtioSound(transport)
if err != nil {
    return err
}

// Inspect the device's PCM stream inventory (control-q PCM_INFO).
infos, err := vs.PCMInfo()
if err != nil {
    return err
}

// Configure stream 0 (the output jack on a single-jack device).
params := virtiosound.PCMParams{
    BufferBytes: 4096,
    PeriodBytes: 1024,
    Channels:    2,
    Format:      virtiosound.PCMFmtS16,
    Rate:        virtiosound.PCMRate44100,
}
if err := vs.PCMSetParams(0, params); err != nil {
    return err
}
if err := vs.PCMPrepare(0); err != nil {
    return err
}
if err := vs.PCMStart(0); err != nil {
    return err
}
```

## License

BSD-3-Clause.
