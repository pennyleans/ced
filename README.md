# ced

The cint editor: a small, plain terminal text editor written entirely in cint, which comes with cint. The terminal and the file reach it through cint's `console/1` and `fs/1` services.

- `SPEC.md`: the living specification of ced 0.1. Every rule has an ID and a check.
- `OPEN.md`: open questions.

## Run it

    cint run --frame-arena 67108864 ced.ci -- notes.txt

The first start compiles ced (about 8 s); `cint run` keeps the build, and later starts take about 0.1 s. `--frame-arena` gives ced room for files up to 8 MiB. The keys are on the bottom line.

## License

Licensed under either of Apache License 2.0 or MIT, at your option.
