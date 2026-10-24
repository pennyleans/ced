# ced: open questions

One entry per question: the clause, what is open, and what is proposed. Nothing here is decided until it is ruled on.

## OQ-1. Wide and combining characters (Open)

Clause: TEXT-3. v0.1 gives every code point one column, so East Asian wide characters and combining marks draw later cells off by a column. Needs a width table; its source and size are open.

## OQ-2. What Tab inserts (Proposed: a tab byte)

Clause: KEY-4. A tab byte keeps the file as the user wrote it. Four spaces would match cint's own sources. Proposed: a tab byte in v0.1; a per-file setting later.

## OQ-3. Esc over a slow link (Open)

Clause: KEY-2. Over SSH, a sequence can arrive split right after its ESC and be read as Esc followed by text. To be measured. If it happens, `keys.ci` holds a trailing ESC for one more `console.read` of up to 25 ms before deciding it is Esc.

## OQ-4. Wait granularity of the read loop (Open)

Clause: RUN-3. `console.read` with a 50 ms wait makes the loop wake 20 times a second when idle, and `console.window` is polled after each wake instead of on `SIGWINCH`. To be measured: idle CPU, and how late a resize shows. A longer wait is the fix if either is a problem.
