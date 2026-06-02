# go-deltasync documentation

Pure-Go, cgo-free building blocks for **delta synchronization** — move and store
only the bytes that change between two versions of a file. Every tool is
wire-compatible with its reference implementation, ships as a single static
binary **and** an importable Go package, and is held to 100 % test coverage.

| Tool | What it is | Interops with |
| --- | --- | --- |
| [zsync2](zsync2/index.md) | download only changed bytes over plain HTTP | zsync |
| [rdiff](rdiff/index.md) | rsync signature / delta / patch | librsync |
| [zchunk](zchunk/index.md) | content-defined chunked container + HTTP-Range delta | zchunk (Fedora/DNF) |
| [vcdiff](vcdiff/index.md) | VCDIFF (RFC 3284) delta encode/decode | xdelta3 |
| [bita](bita/index.md) | differential file sync over HTTP (`.cba` archives) | oll3/bita |

Source for every tool lives at [github.com/go-deltasync](https://github.com/go-deltasync).
