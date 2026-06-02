# zsync2

**A pure-Go, cross-platform reimplementation of the zsync delta-update
protocol &mdash; with a path to BLAKE3.**

[github.com/go-deltasync/zsync2](https://github.com/go-deltasync/zsync2) &nbsp;|&nbsp;
[![License: BSD 3-Clause](https://img.shields.io/badge/license-BSD%203--Clause-blue.svg)](https://github.com/go-deltasync/zsync2/blob/main/LICENSE)

zsync downloads only the *changed* blocks of a file from an HTTP server
using a small control file (`.zsync`) that lists weak + strong checksums
for every block. The client compares those against an existing local
seed and fetches only the bytes it can't recover locally. AppImage,
Debian and Ubuntu ISO mirrors all rely on this for delta updates.

`go-deltasync/zsync2` is a single-binary, cgo-free reimplementation
that ships the same on-the-wire format as the C reference, runs on
Linux/macOS/Windows out of the box, and is the staging ground for the
[BLAKE3 strong-checksum proposal](proposal-blake3.md) that
replaces the broken MD4 / SHA-1 hashes the protocol historically
required.

## What you get

| | Where |
| --- | --- |
| Two CLIs &mdash; `gozsync` (client) + `gozsyncmake` (server side) | [Quick start](quickstart.md) &middot; [CLI reference](cli.md) |
| 100 % statement coverage on the protocol package | [Wire compatibility](compat.md) |
| Byte-identical interop with Phipps' C `zsync` and AppImageCommunity's C++ `zsync2` | [Wire compatibility](compat.md#what-we-test-against) |
| Roadmap for BLAKE3 + modern integrity guarantees | [BLAKE3 proposal](proposal-blake3.md) |

## Install

```bash
go install github.com/go-deltasync/zsync2/cmd/gozsync@latest
go install github.com/go-deltasync/zsync2/cmd/gozsyncmake@latest
```

Pre-built binaries for Linux/macOS/Windows on amd64 and arm64 are
published on every tag &mdash; see the
[Releases page](https://github.com/go-deltasync/zsync2/releases).

## The 30-second demo

```bash
# server side: hash the file and produce the control file
gozsyncmake --url https://example.com/file.bin file.bin
#   wrote file.bin.zsync (1234 blocks of 4096 bytes, total 5050880 bytes)

# client side: reconstruct using a seed
gozsync --input ./old-version --output ./new-version \
        https://example.com/file.bin.zsync
#   zsync: target "file.bin", 1234 blocks of 4096 bytes (5050880 bytes total)
#   seed scan: matched 1198/1234 blocks in 142ms
#   need to fetch 36 blocks (147456 bytes) in 4 ranges
#   wrote ./new-version (5050880 bytes)
```

Full walk-through with a three-terminal end-to-end test on the
[Quick start](quickstart.md) page.

## What's coming

The active design discussion is the [BLAKE3 proposal](proposal-blake3.md):
swap MD4 (broken since 2004) and SHA-1 (broken since 2017) for BLAKE3
across the wire format, keep the legacy `.zsync` path readable forever,
and ship a `.zsync2` alongside it for clients that can use it.

Implementation is in flight on the `main` branch behind a
`gozsyncmake --format zsync2` flag.

## About go-deltasync

This organization hosts cross-platform Go implementations of file-level
delta-synchronization protocols. The common thread: single static binary,
no cgo, and wire compatibility with a canonical reference implementation.

| Project | Family | What it does |
| --- | --- | --- |
| **zsync2** | rsync-over-HTTP | Fetch only changed blocks of a file from a plain HTTP server, using a `.zsync` control file. [Docs &raquo;](quickstart.md) |
| **rdiff** | rsync local (librsync) | Compute a standalone signature/delta/patch between two local file versions, byte-compatible with the C `rdiff`. [Docs &raquo;](../rdiff/quickstart.md) |

More protocols (a `casync`-style chunker, a standardized VCDIFF codec,
&hellip;) may land here as they get the same no-cgo, cross-platform treatment.

[Contributing](../contributing.md) lays out the two non-negotiables for
anything that ships here: wire compatibility against a canonical
reference, and no cgo.
