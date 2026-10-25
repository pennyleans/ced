# ced

The cint editor: a small, plain terminal text editor written entirely in cint, which comes with cint. The terminal and the file reach it through cint's `console/1` and `fs/1` services.

- `SPEC.md`: the living specification of ced 0.1. Every rule has an ID and a check.
- `OPEN.md`: open questions.

## Run it

    cint build --app --frame-arena 67108864 --out build ced.ci
    build/ced notes.txt

Build once, then run with no flag. The frame arena is baked in at build time, so ced takes files up to 8 MiB. The keys are on the bottom line.

## Verify a download

`verify.ci` is a separate program, `ced-verify`, that confirms the files a release's
`SHA256SUMS` lists match what is on disk. It is one job, in cint: it reads
`SHA256SUMS`, hashes each file it names, and refuses any that is missing, too large, or
whose digest differs. It never writes and it reads only the directory it is pointed at,
which is the grant it declares (`fs.read("/arg1")`).

It checks content, not identity. Verify the list's signature with `gpg` or `pacman`
first; `ced-verify` then proves the files are exactly what that signed list names. Its
receipt prints the SHA-256 of `SHA256SUMS` itself, so a run can be tied to the exact
list that was signed.

    cint build --app --out build verify.ci
    build/verify SHA256SUMS

Run it from the directory that holds `SHA256SUMS` and the release files. It prints one
line per entry (`ok`, `mismatch`, `missing` or `refused`), then a receipt, and exits
nonzero unless every entry is `ok`.

## License

Licensed under Apache-2.0 OR MIT.
