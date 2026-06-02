# zchunk

A pure-Go, cross-platform toolkit for the [zchunk](https://github.com/zchunk/zchunk)
file format — content-defined chunking with per-chunk checksums, enabling
**HTTP-Range delta updates** (as used by Fedora/DNF). Interoperable with the
reference `zck` / `unzck` tools.

Repository: [github.com/go-deltasync/zchunk](https://github.com/go-deltasync/zchunk)

```bash
go install github.com/go-deltasync/zchunk/cmd/zchunk@latest
```

## Quick start

```bash
# create a .zck (optionally train + embed a zstd dictionary)
zchunk create -i disk.img disk.img.zck
zchunk create --dict -i disk.img disk.img.zck

# delta-download an update: fetch only the chunks missing locally
zchunk download --header https://example.com/disk.img.zck.header \
  --local old.img.zck https://example.com/disk.img.zck -o new.img.zck
```

## Library

```go
import "github.com/go-deltasync/zchunk"

comp, _ := zchunk.CompressChunk(zchunk.CompressionZstd, nil, data)
// Builder writes .zck files; ReadLead/ReadPreface/ReadIndex parse them;
// PlanDelta + DownloadDelta over a RangeReader perform delta downloads.
```
