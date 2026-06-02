# Contributing

PRs welcome on any of the org's repositories. Two rules:

## 1. Wire compatibility is non-negotiable

Every protocol we ship has a canonical reference implementation
(zsync → Phipps' C, rsync → Tridgell's C, etc.). The compat suite in CI
verifies our output is byte-identical to what the reference would emit
for the same input, and vice versa. **If your change makes the compat
suite skip, that's the same as failing it.** If you genuinely need a
behaviour the reference doesn't have, that's a new tool, not a change to
the existing one.

## 2. No cgo, no platform-specific syscalls

Every project here builds clean on Linux, macOS, and Windows out of the
box. That means:

- stdlib + `golang.org/x/...` only, no cgo bindings to C libraries
- no `unsafe` for layout tricks, only for genuine zero-copy hot paths
  with a comment explaining why
- if you need a syscall that's not in `os` / `io` / `syscall`, that's a
  signal to redesign

## Boring practical stuff

- Run `go test -race ./...` and `go vet ./...` before pushing.
- Run `go test -tags=compat ./...` if you have the reference tools
  installed (`apt install zsync`, or build the C source); CI runs it
  unconditionally.
- Coverage on protocol packages stays at ≥99 % of statements; CI gates
  PRs on this. The CLI wrappers in `cmd/` are exempt — they're thin glue
  and the integration tests cover them.
- One concern per commit. The conventional-commit style isn't strictly
  required but is the norm — `fix(parser): ...`, `feat(cli): ...`, etc.

## Filing bugs

Use GitHub Issues on the affected repo. For protocol-correctness bugs,
include:

- the input that triggers the issue (or a `xxd` excerpt if it's binary)
- the output of the reference implementation on the same input
- the output of our tool on the same input

That's the format the compat suite expects too, so it doubles as the
regression test we'll add when fixing.
