# vcdiff

A pure-Go, **xdelta3-interoperable** implementation of the VCDIFF
([RFC 3284](https://www.rfc-editor.org/rfc/rfc3284)) delta format. Standard wire
layout, so deltas interoperate with `xdelta3 -S` in both directions. Encode is at
parity with the C reference, decode a touch faster, delta slightly smaller.

Repository: [github.com/go-deltasync/vcdiff](https://github.com/go-deltasync/vcdiff)

```bash
go install github.com/go-deltasync/vcdiff/cmd/vcdiff@latest
```

## Quick start

```bash
vcdiff encode -s old.bin new.bin patch.vcdiff   # delta against a dictionary
vcdiff decode -s old.bin patch.vcdiff out.bin
cmp new.bin out.bin                             # identical

vcdiff encode new.bin patch.vcdiff              # no source: pure compression
vcdiff decode patch.vcdiff out.bin
```

## Library

```go
import "github.com/go-deltasync/vcdiff"

delta := vcdiff.EncodeBytes(source, target)
var out bytes.Buffer
_, _ = vcdiff.Decode(source, bytes.NewReader(delta), &out) // out == target
```

For batch workloads reuse a `vcdiff.NewEncoder()` to amortize allocation.
