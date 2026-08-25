# CLAUDE.md — packaging-count

Standing rules for working in this repo. Read `HANDOFF.md` alongside this file
before starting a ticket — `HANDOFF.md` is the living project record (ticket
list, what's done, open decisions, blockers); this file is the stuff that
doesn't change ticket to ticket.

## What this app is

A single-file app: everything — HTML, CSS, JS — lives in `index.html` (no
build step, no bundler, no package manager, no test suite). Supporting files:
`store_reference_band_packaging.json` (T10 reference band data) and
`.claude/launch.json` (dev server config).

There is a sibling repo, `bakery-count`, that shares structure and some
conventions (same field shapes for location classification, similar outlier
guard) but is a **separate codebase with its own `HANDOFF.md`**. Changes here
never touch that repo, and vice versa, unless a ticket explicitly says so —
each repo's session stays in its own working directory.

## Guardrails

1. **`DB_ROOT` must be `''` before every commit.** Both apps talk to
   PRODUCTION Firebase from localhost — no emulator, no staging project. Set
   `DB_ROOT = 'demo'` only for local write-path testing, and revert to `''`
   before committing. Never commit `DB_ROOT = 'demo'`. Note: this repo's
   `demo/` root has no working Firebase security rules, so writes there
   silently fall back to localStorage — see `HANDOFF.md`'s guardrails section
   for the details of that gap.
2. **Verify by serving locally, not `file://`.** `file://` will not work (the
   app needs a real origin for Firebase). Use the `packaging-t4` config in
   `.claude/launch.json` (`python -m http.server 8001`), or equivalent.
3. **One commit per logical change, no partial writes.** Bulk operations
   validate fully and commit whole, or reject and commit nothing.
4. **Never guess a UOM/pack value or misclassify a location.** If source data
   is ambiguous, get a real answer rather than assuming — see `HANDOFF.md`'s
   T4 entry for how past ambiguous cases were resolved and documented.
5. **Item visibility comes from the embedded `ITEMS_DATA` array**, merged with
   Firebase's `items/` node via `ensureItemsSeeded()`, which seeds once and
   skips forever after. Editing the embedded array alone does not reach
   production after the first seed.
6. **Every write path gets a log entry** — who, when, and `source` (`'ui'` /
   `'excel-import'` / `'admin-override'`).
7. **Record *why*, not just *what*, for any decision that isn't obvious from
   the diff.** A constant flip or a config change with no reasoning attached
   just looks arbitrary to the next person who reads it — put the reasoning in
   the code comment and in `HANDOFF.md`.

## Where to look for standing decisions

`HANDOFF.md` has three sections worth checking before assuming something is
still open: "Decisions closed on 25 Aug 2026" (and any later "Follow-up call"
sections dated after it), "Departures from the original plan doc", and "What
still blocks T8/T9". Don't re-raise a closed decision without a new
instruction from the project owner.
