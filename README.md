# ced

The cint editor: a small, plain terminal text editor written entirely in cint, which comes with cint. The terminal and the file reach it through cint's `console/1` and `fs/1` services.

- `SPEC.md`: the living specification of ced 0.1. Every rule has an ID and a check.
- `OPEN.md`: open questions.

## Run it

    cint build --app --frame-arena 67108864 --out build ced.ci
    build/ced notes.txt

Build once, then run with no flag. The frame arena is baked in at build time, so ced takes files up to 8 MiB. The keys are on the bottom line.

## License

Licensed under either of Apache License 2.0 or MIT, at your option.
