# Quick start

## Install

```bash
go install github.com/go-deltasync/zsync2/cmd/gozsync@latest
go install github.com/go-deltasync/zsync2/cmd/gozsyncmake@latest
```

Or grab a pre-built binary from the [Releases page](https://github.com/go-deltasync/zsync2/releases).

## Make a `.zsync` for a file you serve

```bash
gozsyncmake --url https://example.com/file.bin file.bin
# wrote file.bin.zsync (NNN blocks of 4096 bytes, total NNNNNN bytes)
```

Serve both `file.bin` and `file.bin.zsync` from any static HTTP server
(nginx, Caddy, `python3 -m http.server`, S3 — anything that honors
`Range:` requests).

## Fetch an update on the client side

```bash
gozsync --input ./old-version --output ./new-version https://example.com/file.bin.zsync
```

- `--input`: a local copy of the older version (the seed). Optional — if
  you don't have one, gozsync downloads the whole file.
- `--output`: where to write the result. Defaults to the `Filename:`
  embedded in the `.zsync`.

Output:

```
zsync: target "file.bin", 1234 blocks of 4096 bytes (5050880 bytes total)
seed scan: matched 1198/1234 blocks in 142ms
need to fetch 36 blocks (147456 bytes) in 4 ranges
fetching from https://example.com/file.bin
wrote ./new-version (5050880 bytes)
```

## End-to-end test in three terminals

```bash
# terminal 1: prepare files + .zsync, serve them
echo "version 1 content" > target.bin
echo "version 2 content (slightly different)" > seed.bin
gozsyncmake target.bin
python3 -m http.server 8000

# terminal 2: client reconstructs
gozsync --input seed.bin --output out.bin http://localhost:8000/target.bin.zsync

# verify
diff target.bin out.bin && echo OK
```

## What this protocol is good for

- **Distro mirrors**: Debian, Ubuntu, AppImage all use zsync for ISO and
  AppImage updates.
- **Container image diffs**: when the same image differs by a few hundred
  KB across versions, zsync transfers only those KB.
- **Backup / replication**: any time you'd reach for rsync but the source
  is HTTP-served instead of SSH-reachable.
