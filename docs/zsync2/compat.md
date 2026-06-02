# Wire compatibility with upstream zsync

The whole point of `go-deltasync/zsync2` is that the `.zsync` it reads and
writes is **byte-for-byte identical** to what the upstream C/C++
implementations produce. This page is the receipt.

## What we test against

Two reference implementations exist in the wild:

| Implementation | Repo | Language | Status |
| --- | --- | --- | --- |
| `zsync` (Phipps) | <https://github.com/cph6/zsync> | C | the original, ships in Debian/Ubuntu as `zsync` |
| `zsync2` (AppImageCommunity) | <https://github.com/AppImageCommunity/zsync2> | C++17 | the rewrite, ships standalone |

The two share the on-the-wire format byte-for-byte, so it's enough to test
against one of them. We test against the Debian `zsync` package because:

- it's the most widely deployed
- it's the upstream that defines the format
- it installs in CI with `apt-get install -y zsync` — no five-minute
  AppImageCommunity build

If the compat test passes against `zsync`, it passes against `zsync2`.

## The compat test

`internal/zsync/compat_test.go` runs with the `compat` build tag:

```bash
go test -tags=compat ./internal/zsync
```

It does four round trips:

1. **`gozsyncmake` → `zsync`** — we produce a `.zsync`, the C client
   reconstructs the target using only the seed + HTTP server, output
   must be byte-identical
2. **`zsyncmake` → `gozsync`** — same in reverse: upstream produces the
   `.zsync`, we reconstruct
3. **Parse stability** — read a C-produced `.zsync` into our `ControlFile`,
   re-serialize it, diff against the original bytes
4. **Bit-flip detection** — flip one byte in the seed, verify both
   implementations notice the mismatch and re-fetch the affected block

All four pass at every commit; CI gates merges on this. If `zsync` /
`zsyncmake` are not on `$PATH` (local development on macOS, say) the suite
skips cleanly rather than failing.

## CI

The [`compat` workflow](https://github.com/go-deltasync/zsync2/blob/main/.github/workflows/compat.yml)
runs on every push and PR to `main`. It:

1. Installs `zsync` from Ubuntu's apt repo
2. Runs `go test -tags=compat -race ./...`
3. Fails the PR if any of the four round trips diverge by a byte

## What's covered as of this release

- **Multi-URL failover** — `ResolveTargetURL` returns the full list of
  `URL:` entries embedded in the `.zsync`, and `FetchBlocksMulti` walks
  them. Network errors / `5xx` / `404` advance to the next URL; any
  other `4xx` fails fast so a misconfigured URL list doesn't burn
  through every backup on the same problem. Once a URL accepts a Range
  request we stick to it for the rest of the missing blocks.
- **`Z-Map2` reader** — the header parser recognises `Z-Map2:` and
  `Z-URL:` and decodes the per-entry deltas wire format (with the
  `0x8000 NOTBLOCKSTART` flag). The parsed restart-point table is
  exposed at `ControlFile.ZMap`. `ResolveCompressedURLs` is the
  `Z-Map2`-side counterpart of `ResolveTargetURL`. Byte-exact round
  trip via `Write` is preserved.
- **`Z-Map2` + `zsync2: 1.0`** — explicitly rejected at parse time.
  The [BLAKE3 proposal](proposal-blake3.md) hasn't pinned down random-
  access deflate semantics yet, so combining the two is left as a
  separate spec.

## What's still not covered

- **`Z-Map2` maker**. Producing a `.zsync` indexed against a
  gzip-compressed target requires walking the deflate stream to find
  Huffman-table-reset block boundaries. Go's `compress/flate` doesn't
  expose that. A custom deflate-block walker is feasible (~300-500
  lines) but isn't justified yet — the wild ecosystem of pre-existing
  `.zsync` files we want to *consume* is produced by upstream `zsyncmake`,
  which our parser already reads.
- **`Z-Map2` end-to-end smoke test in CI**. The fetcher side is in
  place but the round-trip test requires a C-`zsyncmake -Z`-produced
  gzip `.zsync` on disk; CI's apt `zsync` package doesn't ship the
  matching `gzip` patches consistently across Ubuntu LTS versions.
  Tracked as a follow-up.
