# CLI reference

Both binaries are built on top of [`cobra`](https://github.com/spf13/cobra),
so `--help` is always available and a `-h` short flag is wired up
everywhere.

## `gozsync`

```
gozsync reads a .zsync control file (locally or over HTTP), reuses any
blocks it can find in the given seed file, fetches the rest via HTTP
Range requests against the URL embedded in the .zsync, and writes the
reconstructed file. The on-the-wire format is byte-compatible with the
original zsync (Colin Phipps) and AppImageCommunity/zsync2.

Usage:
  gozsync [flags] <zsync-url-or-file>

Flags:
  -h, --help            help for gozsync
  -i, --input string    local seed file (older version of the target)
  -o, --output string   output path (default: Filename: from .zsync,
                        then basename of URL)
  -q, --quiet           suppress progress output
```

### Argument

`<zsync-url-or-file>` is required. May be:

- an `http://` or `https://` URL — `gozsync` fetches it
- a local path — `gozsync` reads the file directly

If it's a URL and the .zsync embeds relative `URL:` paths, those are
resolved against the *final* URL after HTTP redirects.

### Exit codes

| Code | Meaning |
| --- | --- |
| 0 | Reconstruction succeeded; SHA-1 of output matches the .zsync header |
| 1 | Any other error (network, IO, hash mismatch, malformed .zsync) |

## `gozsyncmake`

```
gozsyncmake hashes the input file block-by-block and writes a
control file that a zsync client can later use to reconstruct the input
on a target machine while re-using as many shared blocks as possible
from a seed file.

Two on-the-wire formats are supported:

  --format=zsync   (default) classic Phipps zsync: 0.6 with MD4 + SHA-1.
                   Compatible with upstream zsync / zsyncmake.
  --format=zsync2  BLAKE3-capable zsync2: 1.0; output defaults to
                   <input>.zsync2. See the BLAKE3 proposal.

When the input is a .gz file you can pass --z-map (or let auto-detect
fire on the .gz extension) to additionally compute the Z-Map2
restart-point table. The resulting .zsync carries Z-URL: + Z-Map2:
headers so a client can fetch the gzipped target by HTTP byte range
and decompress on the fly.

Usage:
  gozsyncmake [flags] <input>

Flags:
  -b, --blocksize int       block size in bytes; must be a power of two
                            (default: auto)
  -f, --filename string     target filename to embed in the control file
                            (default: basename of input)
      --format string       wire format to emit: zsync (classic, MD4+SHA-1)
                            or zsync2 (BLAKE3) (default "zsync")
  -h, --help                help for gozsyncmake
  -o, --output string       output control filename
                            (default: <input>.zsync or <input>.zsync2)
  -u, --url stringArray     URL the client should fetch the uncompressed
                            target from (may be repeated)
      --z-map               compute a Z-Map2 restart table over the input
                            (input must be a .gz file)
  -Z, --z-url stringArray   URL the client should fetch the gzipped target
                            from (Z-Map2 path; may be repeated)
```

### Block size

When `--blocksize` is `0` (default), `gozsyncmake` picks the same value
the C reference picks: a power of two between 1024 and 32768, sized so
the resulting weak-checksum table has ~10 bits of overhead per block.
Bigger blocks → smaller `.zsync` but coarser change detection.

### URLs

Pass one or more `--url` to embed in the `.zsync`. The client picks the
first one that works (multi-URL failover is not implemented in MVP, but
the format is preserved).

If no `--url` is given, `gozsyncmake` embeds `basename(<input>)` as a
relative URL — useful if you serve the `.zsync` and the target from the
same directory.
