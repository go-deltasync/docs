# rdiff — Wire compatibility

`rdiff` reproduces librsync's on-disk formats exactly, so its files are
interchangeable with the upstream C `rdiff` in both directions.

## What we match

- **Delta format** — magic `0x72730236`, the full librsync command set:
  inline LITERAL (1..64), LITERAL with 1/2/4/8-byte length, and the 16 COPY
  variants (`0x45`..`0x54`) selecting 1/2/4/8-byte offset and length, terminated
  by END (`0x00`). Integers are big-endian, minimal width.
- **Signature format** — `magic | block_len | strong_len` (three big-endian
  uint32) followed by `weak:uint32 | strong[strong_len]` per block.
- **Weak checksum** — librsync's Rollsum with `RS_CHAR_OFFSET = 31`, digest
  `(s2 << 16) | s1` mod 2^16.
- **Strong checksum** — MD4 (magic `0x72730136`) or BLAKE2b-256 (magic
  `0x72730137`), truncated to `strong_len`.
- **Short final block** — summed over its actual bytes, never zero-padded.

## What we test against

The [`compat`](https://github.com/go-deltasync/rdiff/blob/main/.github/workflows/compat.yml)
CI job installs the upstream `rdiff` (librsync) on Linux and runs two
cross-implementation round-trips (`go test -tags=compat`):

| Test | Path |
| --- | --- |
| Go signature → **C** delta → Go patch | `TestGoSigCDeltaGoPatch` |
| **C** signature → Go delta → **C** patch | `TestCSigGoDeltaCPatch` |

Both must reconstruct the new file byte-for-byte. The tests `t.Skip` when
`rdiff` is not on `PATH`, so the suite stays green locally without librsync
installed.

## Manual cross-check

```bash
rdiff signature old.bin c.sig          # C reference
go run ./cmd/rdiff delta c.sig new.bin go.delta
rdiff patch old.bin go.delta out.bin   # C reference
cmp new.bin out.bin
```

## Known differences

- A short final basis block is matched only when the **new file ends** with
  those bytes; a short block appearing mid-stream is emitted literally. Output
  is always correct, just occasionally less compact than librsync's.
- Delta generation currently loads the target file into memory.
