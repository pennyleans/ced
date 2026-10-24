# ced: the cint editor

Living specification of ced 0.1. ced is a small, plain (modeless) terminal text editor that comes with cint. It edits one file, with undo and redo, search, and nothing else. It is written entirely in cint (`.ci`): the terminal and the file reach it through cint's `console/1` and `fs/1` services. A rule changes in the same commit as the code that implements it, and every rule ID has a check that names it (TEST-1).

Status words: **v0.1** (in the first release), **later** (designed, not in v0.1), **open** (listed in `OPEN.md`).

## 1. Decisions

| ID | Decision | Why |
| --- | --- | --- |
| D-1 | All of ced is cint. The terminal and the file are reached only through the standard service modules `console` and `fs`, called from `ced.ci` alone, with the grants cint derives from what `ced.ci` declares (RUN-1). There is no C and no shell script in this repository; ced starts with `cint run` and nothing else. | ced needs only what the services provide. |
| D-2 | The **core** (every module except `ced.ci`) is a function of its inputs: the same file bytes, window size and input bytes always give the same text, selection and output bytes. The core calls no service and reads no time, environment or randomness; the compiler's effect classes check this, since every core function must come out `pure`. | Every session replays from its input log, and every bug becomes a test case (TEST-2). |
| D-3 | Input bytes are decoded into **keys**, a **keymap** maps keys to **commands**, and commands are functions over the state. Nothing else changes the text. | The keymap is where vim mode and user bindings go later, without a rewrite. |
| D-4 | Rendering has two steps: the view fills a **cell grid** (one code point and one style per cell), then the terminal writer turns the grid into bytes. | Each step can be optimized on its own. |
| D-5 | Capacities are compile-time constants in `state.ci` and `ced.ci` (STATE-2). When an edit would not fit, the core refuses it and says so; it never writes past a table. | Checked arithmetic, no allocation in the core. |
| D-6 | ced is for people new to cint as well as for daily use: the keys are the common desktop ones (Ctrl+S, Ctrl+Z, Ctrl+F), and the bottom line always shows the main keys. | It comes with cint. |

## 2. Tree

```
ced.ci        main(Grants g): the loop and every service call (RUN-3)
state.ci      the editor state, register names, status codes, capacities
buf.ci        the text: a gap buffer of UTF-8 bytes
undo.ci       the undo log
keys.ci       input bytes to keys
keymap.ci     keys to commands
edit.ci       editing commands
cur.ci        caret and selection movement
search.ci     the find prompt and matching
view.ci       the cell grid: text area, status line, key line, prompt
term.ci       the cell grid to ANSI bytes
core.ci       the pure entry points that ced.ci and the tests call (STATE-3)
```

## 3. Build

| ID | Rule | Status |
| --- | --- | --- |
| BLD-1 | The toolchain is `cint`. | v0.1 |
| BLD-2 | `cint run` builds the program on first use and runs it from the cache (RUN-1); `cint build ced.ci` builds it on its own. Outputs go under the build directory, never into the repository. | v0.1 |
| BLD-3 | `cint test .` runs every module's `test` blocks. | v0.1 |
| BLD-4 | Lint: no float types or floating literals and no CR bytes anywhere; no `.c`, `.h` or shell file in the tree. | v0.1 |
| BLD-5 | `ced` is installed beside `cint` by cint's own build. | later |

## 4. State (`state.ci`, `core.ci`)

| ID | Rule | Status |
| --- | --- | --- |
| STATE-1 | The editor state is module state of `state.ci`: the registers `reg I64[256]` and the tables `text`, `undo`, `clip`, `name`, `msg`, `find`, `grid` and `out`. Registers 0 to 15 are fixed (`R_GAP_START` 2, `R_GAP_END` 3, `R_OUT_LEN` 4, `R_STATUS` 5, `R_REQ` 6); the rest are named in `state.ci` and may change between versions. | v0.1 |
| STATE-2 | Capacities in elements: `text` 16,777,216 (a file larger than half of it, 8,388,608 bytes, is refused, TEXT-1); the save buffer 16,777,216; `undo` 589,824 (65,536 records) with 4,194,304 bytes of removed text; `clip` 8,388,608; `name`, `msg` and `find` 256; `grid` 100 x 400 cells (a larger window is drawn clipped to it); `out` 200,000. The large tables are locals of `main` and so storage of cint's frame arena, 46,967,168 elements in all, and a call of `fs.read` or `fs.put` stages a copy of the file's bytes there, so ced is run with `cint run --frame-arena 67108864`. Module-level arrays are kept to 64 KiB each. | v0.1 |
| STATE-3 | `core.ci` exports the pure entry points: `init(rows, cols)`, `load(in U8[] bytes, in U8[] name)`, `step(in U8[] input)` (one chunk; renders a frame into `out`), `resize(rows, cols)` (renders), `text_out(out U8[] dst)` (the text, contiguous; returns its length), `saved(ok)` (renders), and `select(anchor, head)` (tests). Each returns 0 or a status code. `ced.ci` and the tests are their only callers. | v0.1 |
| STATE-4 | `R_REQ` bits, which the core sets and `ced.ci` clears once it has acted on them: 1 save, 2 quit, 4 bell. | v0.1 |

## 5. The program (`ced.ci`)

`ced.ci` does exactly these things, and never interprets keys, text or styles itself.

| ID | Rule | Status |
| --- | --- | --- |
| RUN-1 | `ced.ci` declares `requires console.input;`, `requires console.raw;`, `requires fs.read("/arg1");` and `requires fs.write("/arg1");`, and is run as `cint run --frame-arena 67108864 ced.ci -- FILE`. With no `--grant` line, cint grants the program its declarations: the console, and `/arg1` backed by the directory of FILE, read as `/arg1/<name>`. `cint run` keeps the built program per version of cint and of the sources, so only the first start compiles, about 8 s; later starts take about 0.1 s. Every path ced names is `/arg1/<name>`. | v0.1 |
| RUN-2 | FILE is `main`'s one program argument, `main(Grants g, in U8[_] args)`: a host path and a NUL. Its last part, after the last `/` or `\\`, is the name; an empty name or one over 256 bytes is refused with a message. A name that is not one logical segment (ASCII letters, digits, `_`, `-`, `.`, no spaces, no trailing `.`, no reserved device name) gets no grant, and ced says so. | v0.1 |
| RUN-3 | The loop: `console.raw(g, true)`; read `console.window(g)` and call `init`; `fs.stat` and `fs.read` the file, or start empty on `not_found`, and call `load`; then repeatedly `console.read(g, chunk, 50)` (at most 4,096 bytes; 0 bytes is a timeout), call `step` when bytes arrived, `console.write` `out[0, R_OUT_LEN)`, act on `R_REQ`, and call `console.window` again, calling `resize` when it changed. On quit, `console.raw(g, false)` and return. | v0.1 |
| RUN-4 | Save: `text_out`, then `fs.put(g, "/arg1/<name>", text)`, which replaces the file atomically, then `saved(1)`; any `FsError` is `saved(0)` and the status line names the error. | v0.1 |
| RUN-5 | Errors from `console` (`closed`, `unavailable`) end the program with `main`'s error result; `denied` means a missing grant, which for the file is a name that is not one logical segment (RUN-2). The runtime restores the terminal when the program ends, faults, or is ended (`console.raw`). | v0.1 |
| RUN-6 | `cint run --record` gives a recording of the session that replays byte for byte. A bug report is that recording. | v0.1 |
| RUN-7 | Rescue after a fault (writing the unsaved text somewhere) waits for a way to run code after a fault; the recording of RUN-6 recovers the session meanwhile. | later |

## 6. Text (`buf.ci`, `undo.ci`)

| ID | Rule | Status |
| --- | --- | --- |
| TEXT-1 | The file is bytes and is saved byte for byte as loaded, apart from the edits. A file that is not valid UTF-8, holds a NUL byte, or is larger than half the text capacity (STATE-2) is refused; ced quits with a message and changes nothing. | v0.1 |
| TEXT-2 | Lines end at LF. A CR before an LF is kept and not shown; a lone CR is shown as `^M`. | v0.1 |
| TEXT-3 | Positions are byte offsets on code point boundaries. Each code point takes one column; a tab advances to the next multiple of 4. Wide and combining characters are open (OQ-1). | v0.1 |
| TEXT-4 | The selection is (anchor, head); the caret is the head. Movement keeps a goal column across lines of different lengths. | v0.1 |
| UNDO-1 | Each change is recorded as (position, removed bytes, inserted bytes, selection before, selection after). Records are grouped, and undo and redo act on a whole group. | v0.1 |
| UNDO-2 | One command is one group. Consecutive typed characters on one line form one group until the caret moves by another command, Enter is pressed, or the group holds 64 characters. Consecutive Backspaces are grouped the same way. | v0.1 |
| UNDO-3 | Undo restores the text and selects what it put back; when it only removed bytes, the caret goes to where they began. Redo applies the group again and restores the selection after it. Any new edit clears the redo groups. | v0.1 |
| UNDO-4 | When the log is full, the oldest groups are dropped. An edit larger than the whole log is applied and clears the log, and the status line says so. | v0.1 |
| UNDO-5 | The log keeps a saved mark. The file counts as modified exactly when the current group differs from the mark. | v0.1 |

## 7. Keys (`keys.ci`, `keymap.ci`, `cur.ci`, `edit.ci`)

| ID | Rule | Status |
| --- | --- | --- |
| KEY-1 | `keys.ci` decodes UTF-8 text, C0 controls, and the xterm CSI and SS3 sequences for arrows, Home, End, PgUp, PgDn, Delete and F3, with Shift and Ctrl modifiers. Bracketed paste arrives as one paste key holding the bytes. | v0.1 |
| KEY-2 | A lone ESC at the end of a chunk is Esc. A sequence split across chunks is joined (the decoder keeps a pending prefix in registers). | v0.1 |
| KEY-3 | Movement: arrows, Home and End (line), Ctrl+Home and Ctrl+End (file), PgUp and PgDn (one screen less one line), Ctrl+Left and Ctrl+Right (word). With Shift, each one extends the selection. | v0.1 |
| KEY-4 | Editing: typed text and paste replace the selection. Backspace and Delete remove the selection, or one code point. Enter inserts LF plus the leading spaces and tabs of the current line. Tab inserts a tab (OQ-2). | v0.1 |
| KEY-5 | Ctrl+A selects all. Ctrl+C copies, Ctrl+X cuts and Ctrl+V pastes the internal clip; with no selection, copy and cut take the whole line. | v0.1 |
| KEY-6 | Ctrl+Z undoes; Ctrl+Y and Ctrl+Shift+Z redo. | v0.1 |
| KEY-7 | Ctrl+S saves. Ctrl+Q quits; when the file is modified, the first Ctrl+Q only warns on the status line and a second one in a row quits. | v0.1 |
| KEY-8 | Ctrl+G opens a `line:` prompt and moves to that line (clamped to the file). | v0.1 |
| KEY-9 | An unbound key rings the bell and changes nothing. | v0.1 |

## 8. Search (`search.ci`)

| ID | Rule | Status |
| --- | --- | --- |
| FIND-1 | Ctrl+F opens a `find:` prompt seeded with the last search. Matching is literal and case-sensitive, moves as each character is typed, and starts from the caret. | v0.1 |
| FIND-2 | Enter or F3 goes to the next match and Shift+F3 to the previous one, wrapping at the ends of the file; the status line says `wrapped`. Esc closes the prompt and keeps the current match selected. | v0.1 |
| FIND-3 | With no match, the status line says `not found` and the selection stays where it was. | v0.1 |
| FIND-4 | Case-insensitive search and replace. | later |

## 9. Screen (`view.ci`, `term.ci`)

| ID | Rule | Status |
| --- | --- | --- |
| VIEW-1 | Rows 0 to rows - 3 show the text. The second-to-last row is the status line: the file name, `[+]` when modified, the message, and `line:col` on the right. The last row is the key line, `^S save  ^Q quit  ^F find  ^Z undo  ^Y redo  ^G line`, or the prompt while one is open. | v0.1 |
| VIEW-2 | Long lines are clipped, not wrapped; the view scrolls horizontally to keep the caret visible, and vertically to keep it on screen. Keeping a row of context past the caret is later. | v0.1 |
| VIEW-3 | Styles: plain, selection (reverse video) and status line (reverse video). There is no syntax highlighting in v0.1. | v0.1 |
| VIEW-4 | `term.ci` writes the whole frame each time: hide the cursor, home, the rows with SGR changes only where the style changes, then the cursor at the caret and show it. Only 7-bit ANSI is used; no 256 or 24-bit color. | v0.1 |
| VIEW-5 | Frame diffing, so that only changed cells are written. | later |

## 10. Tests

| ID | Rule | Status |
| --- | --- | --- |
| TEST-1 | `python tests/run.py [--quick]` (in the development tree) builds ced with `cint build`, runs `cint test .`, replays every golden, and prints `ced: N of N pass`. Each check names its rule ID. | v0.1 |
| TEST-2 | The goldens are keystroke-log cases: `id`, `rules` (the IDs it checks), `text`, `size` [rows, cols], `sel`, `chunks` (hex strings), and the expected `text`, `sel` and optionally `screen`, replayed through the core's entry points (STATE-3). | v0.1 |
| TEST-3 | An end-to-end check runs ced under a pseudo-terminal, types a short session, saves, and compares the file. Recorded, not a gate, in v0.1. | v0.1 |
| TEST-4 | A golden changes only together with the rule it checks, and the commit says why. | v0.1 |
| TEST-5 | Timing: 1,000 characters typed at the end of a 1 MiB file, each `step` under 16 ms. Recorded, not a gate, in v0.1. | v0.1 |

## 11. Writing the core

- `cintc` compiles a subset. Prefer flat forms: scalar locals, `I64` indices into tables, and structs of scalars passed `inout` to functions that are not exported. Big buffers are the module state of `state.ci` (STATE-1), not locals.
- A construct `cintc` refuses is recorded with the form used instead. ced never changes the compiler.
- Arithmetic is checked: a position past the end is a fault, not a clamp. Clamp explicitly where a rule says so.
- Dispatch on keys and commands with `switch`, not long `else if` chains.

## 12. Later, in order

1. Toolchain hooks: run `cint check` and `cint run` from ced, and jump to a diagnostic's position.
2. vim mode.
3. `.ci` highlighting.
4. Several files, frame diffing, case-insensitive search and replace, user keymaps.
