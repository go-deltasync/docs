# chunk

Cuts a stream into pieces at boundaries **the content decides**. Pure Go, no
dependencies, `CGO_ENABLED=0`.

Repository: [github.com/go-deltasync/chunk](https://github.com/go-deltasync/chunk)

```bash
go get github.com/go-deltasync/chunk
```

## Why a rolling hash and not a counter

Cut every sixty-four kilobytes, and inserting one byte at the front moves every
later boundary: nothing matches what was stored before, so the whole file is
written and sent again. Cut where a rolling hash over the last few bytes has
some shape, and inserting one byte disturbs the two chunks around it and nothing
else.

That is what makes deduplication and delta transfer work at all — a backup that
keeps only what changed, an archive that fetches only the chunks it lacks, a
content-addressed store where the same bytes are stored once.

The difference is measured rather than asserted.
`TestAnEditDisturbsOnlyWhatIsNearIt` inserts one byte a tenth of the way into
200 kB and counts what survives:

```
958 of 1067 chunks unchanged after inserting one byte at 10%
cutting every 256 bytes kept 0 of 782
```

## Use

```go
// A stream.
c := chunk.New(reader, chunk.Config{})
for {
    offset, piece, err := c.Next()
    if errors.Is(err, io.EOF) {
        break
    }
    // …
}

// Bytes already in hand, for a caller that wants the pieces and nothing else.
pieces := chunk.Cut(chunk.Config{})(data)
```

`Cut` returns the shape a content-addressed store usually asks for — for
instance `go-crdt/crdt`'s blob store, which takes any chunker:

```go
blobs.PutWith("figure.png", data, chunk.Cut(chunk.Config{}))
```

## Configuration

The zero `Config` is BuzHash over a sixteen-byte window, aiming at 64 KiB chunks
between 16 KiB and 16 MiB — [bita](../bita/index.md)'s defaults. `RollSumConfig` is
bita's other hash, over the window it was tuned for.

`Average` sets how many bits of the hash a boundary needs, so it is rounded down
to a power of two; `Min` and `Max` bound what that can produce. A run of one
repeated byte never trips the test, so `Max` is what ends the chunk — without it
a file of zeroes would be a single chunk.

## Where it comes from

The rolling hashes, the seed, the table and the boundary test are
[bita](../bita/index.md)'s, so a stream cut here is cut in the same places bita cuts
it, and an archive written by either is readable by the other. They were written
for bita and lived inside it, where nothing else could reach them. This is the
same code with a name, so the next thing needing content-defined chunks does not
write its own.

## ⛔ The other two rolling sums are not this one

Two more live in this organisation and are **not** interchangeable with it:
`rdiff`'s is librsync's and `zsync2`'s is zsync's. All three are the same
Fletcher family and no two are the same function — different widths, different
initial state, differently packed digests, and zsync adds no offset to a byte
where the other two add thirty-one. On the same eight bytes:

```
chunk.RollSum 0x011c0740
rdiff.Rollsum 0x04d4011c
zsync.Rsum    0x00240078
```

That is not an oversight to tidy up. Each is a constant of a wire format
something else already wrote — a signature file, a `.zsync` header, an archive.
Sharing one would make this package's users agree with each other and one of
those formats unreadable.
