# bita

A pure-Go, cgo-free reimplementation of [oll3/bita](https://github.com/oll3/bita):
differential file synchronization over HTTP. `bita` chunks a file with a
content-defined rolling-hash chunker, stores unique chunks in a compressed
`.cba` archive, and reconstructs ("clones") the file by reusing chunks already
available locally (the *seeds*) and fetching only the rest. Wire-compatible with
the Rust `bita` (RollSum and BuzHash), verified both directions.

Repository: [github.com/go-deltasync/bita](https://github.com/go-deltasync/bita)

```bash
go install github.com/go-deltasync/bita/cmd/bita@latest
```

## Quick start

```bash
bita compress -i disk.img disk.cba

# reconstruct from an archive, reusing a local seed (local path or http(s) URL):
bita clone --seed v1.img https://example.com/v2.cba v2.img --verify-output
# cloned 4194304 bytes (4142202 from seeds, 52102 from archive)

bita info disk.cba
```

## Library

```go
import "github.com/go-deltasync/bita"

var arc bytes.Buffer
_ = bita.Compress(input, &arc, bita.CompressConfig{})
a, _ := bita.OpenArchiveReaderAt(bytes.NewReader(arc.Bytes()))
_, _ = bita.Clone(a, out, bita.CloneOptions{Seeds: []io.Reader{seed}})
```
