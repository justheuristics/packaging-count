# HANDOFF — Counting Apps action plan (packaging-count)

This repo and its sibling `bakery-count` (`github.com/justheuristics/bakery-count`)
are being worked through a 9-ticket action plan in order, one ticket per branch,
one PR per ticket, merged before the next starts. **Read this whole file before
starting T4 (the next applicable ticket here) or any later ticket** — it has
the guardrails, the full ticket list, what's already done, and what's known to
depart from the original plan doc.

If you have the original plan doc (`CLAUDE_CODE_ACTION_PLAN_Counting_Apps.md`),
this file summarizes it plus everything learned while implementing T1/T2 that
the original doc got wrong or didn't anticipate — read this file's departures
section even if you have the original, since some of its stated facts are stale.

## Guardrails — apply to every ticket, not just the one you're on

1. **Both apps talk to PRODUCTION Firebase from localhost.** No emulator, no
   staging project. Set `DB_ROOT = 'demo'` (this repo: `index.html`, currently
   `''`) before any write-path testing, revert to `''` before committing.
   Never commit `DB_ROOT = 'demo'`. **This repo's `demo/` root has no working
   Firebase security rules** — every read/write under it returns
   `PERMISSION_DENIED` and silently falls back to localStorage (there's
   already a `firebaseRulesCard` warning UI in this codebase for exactly this
   — see `setMonthStatusDirect`). The localStorage fallback only caches
   **individual leaf paths**, not subtrees, so any code that reads a parent
   path (e.g. `counts/{month}/{store}`, whole-month reads) can never be
   satisfied from the fallback even after a successful leaf-level write.
   This blocked full live verification of T2 here — see below. If you need
   real demo-mode round-trip testing in this repo, get this fixed first, or
   accept logic-level verification as bakery's T2 did (that was an explicit
   user decision, not a default — ask before assuming it again for a new
   ticket).
2. **Item visibility comes from the embedded `ITEMS_DATA` array in
   `index.html`, merged with Firebase's `items/` node.** `ensureItemsSeeded()`
   seeds `items/` to Firebase **once** and then skips forever if it already
   has data — editing the embedded array alone does not reach production
   after the first seed. T1 discovered this the hard way: the embedded array
   had 11 stale string `packRec` values that a source-only fix didn't
   correct in production; had to fix them via the admin edit UI directly
   (raw Firebase writes were blocked by the harness's auto-mode classifier —
   go through the app's own admin UI for any production Firebase correction,
   don't try to script around that block).
3. **Never guess a UOM/pack value or misclassify a location.** Matches
   bakery's equivalent guardrail. A wrong `packRec`/`packCount` type is a
   data-hygiene issue here (see T1 notes below — it turned out not to be a
   live valuation bug, but treat it as one until proven otherwise).
4. **No partial writes.** Bulk operations validate fully and commit whole, or
   reject and commit nothing.
5. **Every write path gets a log entry** with who, when, and `source`
   (`'ui'` / `'excel-import'` / `'admin-override'`). T2's saves now tag
   `source:'ui'` and `outlierConfirmed:<count>`.
6. **Preserve existing behaviour when porting between bakery and packaging** —
   match bakery's existing output format rather than improving it, since
   these two reports get compared side by side. This repo had **zero**
   estimated-cost disclaimers anywhere before T2 (bakery already had them) —
   T2 swept the gap using bakery's exact disclaimer wording and its
   column-header-embedding technique for exports.
7. **One commit per ticket, stop for review after each 🔴 P0 ticket.**

## Full ticket list

| # | Priority | Scope | Status |
|---|---|---|---|
| T1 | 🔴 P0 | Reject duplicate item codes at load; normalise pack fields | ✅ committed, merged to `main` (both repos) |
| T2 | 🔴 P0 | Outlier variance guard + admin exception queue | ✅ committed on `ticket-2-outlier-guard` (both repos), **not yet pushed** |
| T3 | 🟢 P1 | Remove plaintext password column from bakery admin UI | N/A — **bakery only**, nothing to do here (this repo has no store admin UI at all: `STORES_DATA` is a hardcoded source array, never Firebase-backed) |
| T4 | 🟢 P1 | Location type + date-effective open/closed status (**both apps**) | ⬜ not started — this repo carries the harder half, see departures below |
| T5 | 🟢 P1 | Export store submission status to Excel (**this repo** — bakery already has this) | ⬜ not started — reference implementation is bakery's `exportStoreStatusExcel()`; this repo's `exportRowsToExcel()` helper (already loaded, unused by the dashboard) is what to reuse |
| T6 | 🟢 P1 | Price / priceUom / priceEffectiveFrom on bakery item master | N/A — **bakery only** |
| T7 | 🟢 P1 | Align Thai calendar display (BE) across both apps | ⬜ not started — this repo's `thaiMonthLabel`/`thaiDate`/`fmtDateTime` are currently CE, need `+543` |
| T8 | 🟡 P2 | Price list bulk import with preview-diff-confirm | ⛔ **blocked** on open question Q3 |
| T9 | 🟡 P2 | Stock-take Excel upload with validation gate | ⛔ **blocked** on open question Q4 |

Suggested commit sequence (unchanged): T1 → T2 → T3(bakery-only, skip here) →
T4 → T5 → T6(bakery-only, skip here) → T7 → **stop, await Q3/Q4** → T8 → T9.

## What's done

### T1 — duplicate-code validator + pack-field fixes (merged)
`scanItemMasterIssues()` in `ensureItemsSeeded()`/`loadItemsFromDB()`, surfaced
as a persistent `#masterDataAlert` banner + a detailed card on Overview + a
"รหัสซ้ำ" chip on affected entry rows. Fixed the source `ITEMS_DATA`: coerced
11 string `packRec` values to numbers, de-duplicated the retired-category
`0160220151Y` row (282 items now, was 283). Left the 5 `Non-Cat` FRESH rows
flagged, not touched — for the Buyer to assign real codes (open question Q6).

**Also fixed in production Firebase** (via the admin edit UI, not a raw
write — see guardrail 2 above): the same 11 `packRec` values were stale
strings in the live `items/` node, unreachable by the source fix alone.
Verified: `packRec` itself is never used in arithmetic anywhere (only
displayed via `fmtNum`, which already coerces) — this was a type-hygiene fix,
not a correction of a live valuation bug. `packCount` (the field that *is*
used in cost math) was already `Number(...)`-wrapped at every call site.

**Also discovered, not fixable without guessing**: production Firebase's
`items/` node had already silently collapsed the 5 `Non-Cat` variants to 1
survivor (code `Non-Cat`, no.136, ฿2,850) and the 2 `0160220151Y` variants to
1 (no.41, ฿750), via object-key overwrite during `ensureItemsSeeded()` — the
exact failure T1 describes, already realized live before this work started.

### T2 — outlier variance guard + admin exception queue (committed, not pushed)
Mirrors bakery's T2, adapted to this app's category+code keying
(`FRESH`/`TRANSFER`/`NONFRESH` × item code — no shared module system, logic
is duplicated not shared). `OUTLIER_FACTOR = 10`, `loadOutlierContext()`
(fire-and-forget from `loadMonthData()`, tracked via
`OUTLIER_CONTEXT_PROMISE` so `saveCategory()` always awaits it before
computing flags), `evalOutlier()`, `computeOutlierFlag(cat, code, rec)`.
Two-phase `saveCategory(confirmedFlags)`: first call computes flags on
changed lines, shows a modal if any exist; confirm re-calls with the flagged
list, stamps `confirmedBy`/`confirmedAt`/`flagReason`. Admin dashboard gains
the exception card (via `computeAndPersistOutlierExceptions(month)`, wired
through `loadDashboard()`'s module-level `DASH_EXCEPTIONS`), persisting
`stats/{key}/itemMedian`.

**Verification is partial** — confirmed via direct calls into the real
`computeOutlierFlag()`/`saveCategory()` functions (manually seeding
`PRIOR_MONTH_TOTALS` since the demo-Firebase-rules gap above blocks a real
two-month save/reload cycle), and confirmed T1's validator still works
post-T2. **Never click-tested**: the exception card actually rendering with
real flagged data in the dashboard UI. Treat the admin card as logic-reviewed
against bakery's proven equivalent, not click-verified. This was an explicit,
discussed tradeoff — not an oversight — but if you're picking up T4/T5 and
touch the dashboard, it's worth a real look while you're in there.

## Departures from the original plan doc — read before starting T4

These were verified against actual code/data while implementing T1/T2 and
change how T4 should be approached here:

1. **This repo's 208 locations, classified by name pattern
   (`คลังสินค้า`→DC, `เอฟซี`→FC, `Dummy`→DUMMY, `ร้านค้าเสมือนจริง`→VIRTUAL,
   `สยามโฟรเซ่น`→FROZEN)**: DC 19, FC 5, DUMMY 1, VIRTUAL 1, FROZEN 9,
   STORE 173. **Two locations need an explicit override, not just the
   pattern**: loc `806` ("Ningbo Beicang (Tmall)") matches no Thai pattern
   and would silently fall through to `STORE` — should be `VIRTUAL`. Loc
   `402` is named `คลังสินค้าเอเอฟซี-บางนา` → classifies as `DC` by its own
   name, which **contradicts** the original plan doc's listing of it as FC —
   trust the name, flag the discrepancy in the commit message, don't just
   follow the doc.
2. `528`/`801`/`804` (Siam Frozen-adjacent codes near others already
   classified) should stay `STORE` and be classified-but-not-excluded
   pending open question Q1 — put the exclusion behind a one-line config
   constant so flipping it later is trivial.
3. `STORES_DATA` in this repo is a **hardcoded source array**, never
   Firebase-backed (confirmed: root-level Firebase reads return 401/denied
   for anything outside `items/`). T4's "never delete a location" already
   holds here — there's no delete mechanism to replace. All the T4 work in
   this repo is: add the lifecycle fields to `STORES_DATA` entries, build
   `isCountableAt(loc, ym)`, apply it to `computeCategorySums`/
   `loadOverviewStats`/`renderDashboardResult` and all exports, add the
   admin "แสดงเฉพาะสาขา / แสดงทั้งหมด" toggle.
4. `isCountableAt` needs to match bakery's exact semantics (date-range is
   authoritative for as-at evaluation, `status` is just the current label) —
   see bakery's `HANDOFF.md` if it still has T4 notes, or the shared plan
   file referenced by whichever Claude Code session did T2, for the intended
   shared shape:
   `{locNo, name, username, locationType, status, effectiveFrom, effectiveTo, statusNote}`.

## Before starting T4 (or T5, if doing that first)

1. Confirm T1 and T2 are merged to `main` (check `git log origin/main`).
2. `git checkout -b ticket-4-location-status main` (or `ticket-5-...` if
   sequencing differently — T4 and T5 don't depend on each other within this
   repo, but T5's export needs `isCountableAt` from T4 to exclude
   DC/FC/dummy locations correctly, so do T4 first if at all possible).
3. This ticket touches the write path (location field migration) — set
   `DB_ROOT = 'demo'` for testing, revert before committing. Given
   guardrail 1's demo-rules gap, decide up front whether to test with the
   `demo` root (logic-only, no real Firebase round trip) or ask the project
   owner to fix packaging's Firebase rules first.
4. Delete this section (and update the "What's done" table above) once T4
   is merged, so this file stays a living resume point instead of drifting
   stale. Don't delete the guardrails/departures sections — they stay
   relevant for the rest of the plan.
