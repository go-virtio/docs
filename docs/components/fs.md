# fs — virtio-fs

`github.com/go-virtio/fs` is a pure-Go virtio-fs (FUSE-over-virtio) guest
driver for the standard PCI-bound device (VID 0x1AF4, DID 0x105A — device
type 26). It implements the modern-transport (Virtio 1.0+) init sequence,
the request virtqueue, and the FUSE-over-virtio wire framing. CGO=0, no
architecture-specific assembly, 100% statement test coverage.

## Scope

Like [`blk`](blk.md), this package owns device bring-up, a request
virtqueue, and the on-the-wire request format. For virtio-fs the request
is a FUSE message carried as a descriptor chain (Virtio 1.2 §5.11.6):

```
readable descriptors : [ struct fuse_in_header  | op-specific in-args  ]
writable descriptors : [ struct fuse_out_header | op-specific out-args ]
```

The driver exposes a **read-write mount closure**, each op cited from
Linux `include/uapi/linux/fuse.h`.

| Side | Ops |
|------|-----|
| Read | `Init`, `Lookup`, `GetAttr`, `Open`, `Read`, `Release`, `Forget`, `Destroy` |
| Write | `OpenRW`, `Write`, `Create`, `Mkdir`, `Mknod`, `Symlink`, `Link`, `SetAttr`, `Unlink`, `Rmdir`, `Rename`, `Fsync`, `Flush` |

`SetAttr` covers truncate (`FattrSize`), chmod (`FattrMode`), chown
(`FattrUID`/`FattrGID`) and utimes (`FattrAtime`/`FattrMtime`) via the
`fuse_setattr_in.valid` mask. `FUSE_WRITE` is the only op whose request
is a three-region descriptor chain (readable header + data, writable
reply).

!!! note "Validated against a real virtiofsd"
    v0.2.1 fixes a `fuse_attr` struct-size bug found by real-virtiofsd
    validation — the kind of bug a fake-device unit test cannot catch.

## License

BSD-3-Clause.
