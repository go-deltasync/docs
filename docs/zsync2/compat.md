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
- **`Z-Map2` maker** — producing a `.zsync` indexed against a
  gzip-compressed target. A self-contained deflate-bitstream walker
  (`internal/zsync/deflate_walker.go`) finds every block-boundary /
  Huffman-reset point and records the `(compressed-bit-offset,
  uncompressed-byte-offset)` table, since Go's `compress/flate` exposes
  no boundary hooks. Driven by `gozsyncmake --z-map` (auto-enabled for
  `.gz` input) and the `MakeWithZMap2` / `EncodeZMap2` API.
- **`Z-Map2` end-to-end tests** — `TestCompatZMap2OurMakeRoundTrip`
  exercises our maker + our reader, and `TestCompatZMap2UpstreamMakeOurApply`
  builds the index with the C `zsyncmake -Z` and applies it with our
  client. The latter skips cleanly when the apt-installed `zsyncmake`
  was compiled without the gzip-aware `-Z` path (it varies across Ubuntu
  LTS versions).
- **`Z-Map2` + `zsync2: 1.0`** — explicitly rejected at parse time.
  The [BLAKE3 proposal](proposal-blake3.md) hasn't pinned down random-
  access deflate semantics yet, so combining the two is left as a
  separate spec.

## What's still not covered

- **Client-side `Z-Map2` fetch.** The maker emits `Z-URL:` + `Z-Map2:`
  and the client parses the restart-point table into `cf.ZMap`, but the
  download path — *fetch a gz byte range, reset the inflater at a
  restart bit-offset (priming the 32 KiB back-reference window),
  decompress just enough to cover a missing block* — is not yet wired
  into `gozsync`. A client pointed at a `Z-Map2` `.zsync` whose `URL:`
  is a gz endpoint currently falls back to downloading the gz whole-file.
- **`Recompress`.** The header round-trips through `Write`, but the
  client does not act on it.
- **BLAKE3 + `Z-Map2`.** Intentionally unspecified: the parser rejects
  the combination loudly and the maker refuses `--z-map --format=zsync2`,
  pending the [BLAKE3 proposal](proposal-blake3.md) pinning down the
  random-access deflate semantics.
