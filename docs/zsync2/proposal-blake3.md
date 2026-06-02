# Proposal: BLAKE3 strong checksum

**Status:** Draft &mdash; comments welcome on
[github.com/go-deltasync/zsync2/discussions](https://github.com/go-deltasync/zsync2/discussions).
**Target version:** zsync2 wire format `1.0` (current shipping ships
the classic Phipps wire format `0.6`).

## Problem

The zsync wire format requires two cryptographic hashes:

| Where used | Algorithm | Status |
| --- | --- | --- |
| Per-block strong checksum (after rolling-checksum match) | **MD4** | Broken since 2004 (Wang, Lai, et al.); 2^6 work for a collision on a modern CPU |
| Whole-file integrity at end of reconstruction | **SHA-1** | Broken since 2017 (Stevens et al., SHAttered) |

For the typical zsync deployment &mdash; an Ubuntu mirror serving ISO
diffs &mdash; this is "good enough", because the trust anchor is the
*TLS-served `.zsync` control file*, not the hashes inside it. But the
choice constrains every threat model that goes beyond accidental
corruption.

Concretely:

- A constrained adversary who can write to the **seed file** (CI cache
  poisoning, evil-maid on a laptop, a malicious package
  pre-installation) can craft MD4 collisions targeted at specific
  blocks of a known target. The matcher will accept the colliding
  bytes; the seed-to-target copy then carries the adversary's payload.
  The file-wide SHA-1 catches it &mdash; *unless* the adversary also
  crafts a SHA-1 collision for the whole reconstructed file (more
  expensive but no longer rocket science).
- Anybody who simply wants to **package zsync2 in a security-conscious
  distro** has to wave hands at MD4. With BLAKE3 there's no hand-waving.

This proposal upgrades both hashes to **BLAKE3** without breaking
existing `.zsync` consumers.

## Why BLAKE3 specifically

- **Speed**: ~6 GB/s single-threaded on modern x86-64 (Phipps' MD4 sits
  around 700 MB/s), faster on aarch64-NEON, near memory-bandwidth on
  AVX-512. The `gozsyncmake` step is currently MD4-bound; BLAKE3
  removes that floor.
- **Tree structure**: lets us parallelise per-block hashing on multi-core
  makers for free, and lets future extensions key-derive per-block.
- **Modern security margin**: 256-bit collision resistance vs MD4's
  effective 0.
- **Truncation is well-defined**: BLAKE3's output is a 32-byte stream;
  truncating to the per-block `Hash-Lengths` width keeps the same
  birthday-bound properties as truncating SHA-256. zsync's truncation
  policy ports verbatim.
- **Stdlib-adjacent**: Go has [`lukechampine.com/blake3`][b3-go] (pure
  Go, vendor-free), Rust has the official `blake3` crate, C/C++ ship
  the reference implementation at <https://github.com/BLAKE3-team/BLAKE3>.

[b3-go]: https://github.com/lukechampine/blake3

## Wire format changes (proposed)

The current header block looks like:

```
zsync: 0.6
Filename: file.bin
MTime: Wed, 01 Jan 2025 00:00:00 +0000
Blocksize: 4096
Length: 5050880
Hash-Lengths: 2 2 4
URL: https://example.com/file.bin
SHA-1: 0e23...
<empty line>
<binary block table>
```

The new format adds two headers and bumps the version:

```
zsync2: 1.0                            ← new magic; was "zsync: 0.6"
Filename: file.bin
MTime: Wed, 01 Jan 2025 00:00:00 +0000
Blocksize: 4096
Length: 5050880
Hash-Lengths: 2 2 16                   ← checksum_bytes widened
Hash-Algorithm: BLAKE3                 ← NEW
URL: https://example.com/file.bin
File-Hash: BLAKE3:5b46...              ← NEW (replaces SHA-1)
SHA-1: 0e23...                         ← OPTIONAL legacy fallback
<empty line>
<binary block table: same layout, BLAKE3 bytes instead of MD4>
```

Two new keys, one new magic:

1. **`zsync2:` magic line** &mdash; replaces `zsync:`. Old Phipps `zsync`
   parsers fail with "not a zsync file" and exit cleanly; no risk of a
   stale parser misinterpreting BLAKE3 bytes as MD4.
2. **`Hash-Algorithm:`** &mdash; one of `MD4` (legacy, the default
   if absent) or `BLAKE3`. Future-proofs the field.
3. **`File-Hash:` `<algo>:<hex>`** &mdash; replaces `SHA-1:`. Same role,
   self-describing. A maker that wants compat can emit *both*
   `File-Hash:` and `SHA-1:` so a `zsync2: 1.0` reader uses BLAKE3
   while a hypothetical `zsync: 0.6`-aware downgrade reader sees the
   SHA-1.

The binary block table layout doesn't change. Per block:

```
<rsum_bytes bytes of rolling-checksum tail><checksum_bytes bytes of strong-hash prefix>
```

Only the *meaning* of the strong-hash prefix changes (MD4 truncation
&rarr; BLAKE3 truncation). Same width, same offset.

### Compat matrix

| Maker emits | Reader sees | Behaviour |
| --- | --- | --- |
| `zsync: 0.6` (classic) | Old `zsync` | Works (today) |
| `zsync: 0.6` (classic) | New `gozsync` | Works (we read both) |
| `zsync2: 1.0` (new) | Old `zsync` | **Rejects** with "not a zsync file" &mdash; safe |
| `zsync2: 1.0` (new) | New `gozsync` | Works, uses BLAKE3 |

The deliberate non-symmetry: **upgrading is a one-way decision**. A
maker that emits `zsync2: 1.0` is saying "I want BLAKE3-aware clients
only". To target old clients too, the maker emits *two* control files:
`file.bin.zsync` and `file.bin.zsync2`. Clients that know about the
new format prefer `.zsync2`; old clients pick `.zsync`.

This matches what GnuPG did with the `.asc` &harr; `.sig` split:
filename extension carries the version, content stays clean.

## Cross-implementation strategy

Three implementations to bring along:

1. **`go-deltasync/zsync2`** (this repo): implements both
   `zsync: 0.6` (today) and `zsync2: 1.0` (this proposal). Behind a
   `gozsyncmake --format zsync2` flag initially; new default after one
   release.
2. **`AppImageCommunity/zsync2` (C++)**: upstream PR adding a
   `--format=zsync2` mode. Existing `zsyncmake2` produces classic by
   default, `--format=zsync2` produces new. The bidirectional read
   support is the harder ask, but the BLAKE3 reference C++ is
   well-maintained.
3. **`Phipps zsync` (C)**: maintenance is sparse; we'd land the spec
   only and trust the Debian zsync package to follow if there's demand.

The compat test suite in our CI grows a fourth round trip: maker C++
zsync2-cpp &rlharu; reader gozsync, in both directions, with
`--format=zsync2`. The current four MD4/SHA-1 round trips stay green
&mdash; legacy isn't going anywhere.

## Implementation cost (in `go-deltasync/zsync2`)

About 100&ndash;150 lines added, no removals:

- `internal/zsync/control.go`: parse `zsync2:` line + `Hash-Algorithm:`
  + `File-Hash:`; emit them in `Write` when format is `zsync2`.
- `internal/zsync/rsum.go`: factor `strongHash(blk []byte) []byte` out
  of the MD4-specific code path; add a BLAKE3 implementation. Both
  paths get a unit test against fixed test vectors.
- `cmd/gozsyncmake/main.go`: new `--format` flag with values `zsync`
  (default) and `zsync2`.
- `internal/zsync/compat_test.go`: extra round trips against
  `xxhsum` &mdash; wait, no, against the C++ reference once it lands.
- `go.mod`: adds `lukechampine.com/blake3 v1.x`.

Coverage gate stays at &ge;99 % on `internal/zsync`.

## Open questions

1. **Truncation width.** MD4 zsync files traditionally truncate to 2-4
   bytes per block to keep `.zsync` small. With BLAKE3 we could go
   wider for ~free relative to total file size. The proposal keeps
   the existing `Hash-Lengths:` formula; concretely a 1 GB target
   gets `Hash-Lengths: 2 4 16` &mdash; 16 bytes of BLAKE3 per block,
   128-bit birthday-bound on accidental collision.
2. **Is `File-Hash:` worth being polymorphic?** Could just say
   "BLAKE3" and bake it into the format. Argument for `<algo>:<hex>`:
   if BLAKE4 ships in 10 years we want the upgrade path.
3. **Should the maker emit both files by default?** Probably yes for
   the first major release: `gozsyncmake foo.bin` emits both
   `foo.bin.zsync` and `foo.bin.zsync2`. Drop the legacy after one
   year of telemetry.

## Reference docs

- [BLAKE3 paper](https://github.com/BLAKE3-team/BLAKE3-specs/blob/master/blake3.pdf)
- [Phipps' original zsync paper](https://zsync.moria.org.uk/paper200524/paper.html)
- [Wang et al., "Collisions for Hash Functions MD4, MD5, HAVAL-128 and RIPEMD"](https://eprint.iacr.org/2004/199)

---

If this lands as proposed, the entry in the security section of zsync2's
README changes from *"use MD4 because that's the protocol"* to *"use
BLAKE3 because that's the protocol, with MD4 as a documented legacy
fallback for the year-one transition window"*. That's the goal.
