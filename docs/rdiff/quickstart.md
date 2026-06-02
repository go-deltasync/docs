# rdiff — Quick start

`rdiff` is a pure-Go, **librsync-interoperable** implementation of the classic
rsync signature / delta / patch workflow. It computes a small *delta* between an
old and a new version of a file, so you ship only what changed.

```
signature  BASIS         -> SIGNATURE   weak + strong checksums of every block
delta      SIGNATURE NEW  -> DELTA       instructions to turn BASIS into NEW
patch      BASIS DELTA    -> NEW         apply the instructions
```

Unlike [zsync2](../index.md), which fetches changed blocks over HTTP, `rdiff`
works on **local files** and produces a standalone delta you can transport any
way you like — exactly like the C `rdiff` from librsync, and byte-compatible
with it.

## Install

```bash
go install github.com/go-deltasync/rdiff/cmd/rdiff@latest
```

Pre-built static binaries for Linux/macOS/Windows on amd64 and arm64 are
attached to every tagged release.

## The 30-second demo

```bash
# 1. The party holding the old file publishes a signature (small).
rdiff signature app-v1.bin app.sig          # default: BLAKE2b, 2048-byte blocks

# 2. The party holding the new file uses that signature to compute a delta.
rdiff delta app.sig app-v2.bin app.delta

# 3. Anyone with the old file + the delta reconstructs the new file.
rdiff patch app-v1.bin app.delta app-v2.rebuilt

cmp app-v2.bin app-v2.rebuilt                # identical
```

For a 200 KB file changed in one spot, the delta is a couple of KB instead of
re-shipping the whole file.

## Choosing a hash

The strong checksum defaults to **BLAKE2b-256** (modern librsync default). For
interop with very old signatures, switch to MD4:

```bash
rdiff signature -H md4 app-v1.bin app.sig
```

`-` may be used for the streamable argument of any command (stdin/stdout).
`patch`'s BASIS must be a real file, since COPY commands read it at random
offsets.
