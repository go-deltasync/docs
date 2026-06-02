# rdiff — CLI reference

The binary is built on [`cobra`](https://github.com/spf13/cobra), so `--help`
(and `-h`) is available on every command, and `--version` prints the build.

## `rdiff signature`

```
rdiff signature [flags] BASIS SIGNATURE

Compute the signature of BASIS.

Flags:
  -b, --block-size int    signature block size in bytes (default 2048)
  -H, --hash string       strong-sum hash: blake2 or md4 (default "blake2")
  -S, --strong-size int   truncate strong sum to this many bytes (0 = full digest)
  -h, --help              help for signature
```

The signature lists, per block of BASIS, a 4-byte rolling weak checksum and a
(optionally truncated) strong checksum. Smaller `--block-size` → finer change
detection but a larger signature.

## `rdiff delta`

```
rdiff delta SIGNATURE NEWFILE DELTA

Compute the delta from SIGNATURE's basis to NEWFILE.
```

A rolling weak checksum slides over NEWFILE; weak-sum hits are confirmed by the
strong sum and emitted as COPY commands referencing the basis, while unmatched
runs become LITERALs.

## `rdiff patch`

```
rdiff patch BASIS DELTA NEWFILE

Apply DELTA to BASIS, producing NEWFILE.
```

BASIS must be a regular file — COPY commands read it at random offsets, so it
cannot be a stream.

## Conventions

| | |
| --- | --- |
| `-` argument | stdin/stdout for the streamable I/O of each command |
| Exit code 0 | success |
| Exit code 1 | any error (IO, malformed signature/delta, bad magic) |

## On-disk formats

| File | Magic (big-endian) |
| --- | --- |
| delta | `0x72730236` |
| signature (MD4) | `0x72730136` |
| signature (BLAKE2b) | `0x72730137` |

These are the librsync magics, so files are interchangeable with the C `rdiff`.
See [Wire compatibility](compat.md).
