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
# create a .zck from a file (fixed-size chunks, zstd)
zchunk create disk.img disk.img.zck

# train + embed a zstd dictionary, then create with it
zchunk gen-zdict disk.img disk.dict
zchunk create --dict disk.dict disk.img disk.img.zck

# delta-download an update: fetch only the chunks missing locally,
# reusing an older local copy
zchunk download --local old.img.zck https://example.com/disk.img.zck new.img.zck
```

## Library

```go
import "github.com/go-deltasync/zchunk"

comp, _ := zchunk.CompressChunk(zchunk.CompressionZstd, nil, data)
// Builder writes .zck files; ReadLead/ReadPreface/ReadIndex parse them;
// PlanDelta + DownloadDelta over a RangeReader perform delta downloads.
```
