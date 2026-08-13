# HANDOFF — Counting Apps action plan (packaging-count)

This repo and its sibling `bakery-count` (`github.com/justheuristics/bakery-count`)
are being worked through a 9-ticket action plan in order, one ticket per branch,
one PR per ticket, merged before the next starts. **Read this whole file before
starting T8 or T9** — it has the guardrails, the full ticket list, what's
already done, and what's known to depart from the original plan doc.

If you have the original plan doc (`CLAUDE_CODE_ACTION_PLAN_Counting_Apps.md`),
this file summarizes it plus everything learned while implementing T1–T7 that
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
   T4/T5/T7 here added no new write path (T4/T5 are read-only classification
   and export logic; T7 is display-only), so this gap didn't block anything
   after T2 — but it's still unfixed, and still blocks a real demo-mode
   round-trip test for whatever comes next that does write.
2. **Item visibility comes from the embedded `ITEMS_DATA` array in
   `index.html`, merged with Firebase's `items/` node.** `ensureItemsSeeded()`
   seeds `items/` to Firebase **once** and then skips forever if it already
   has data — editing the embedded array alone does not reach production
   after the first seed.
3. **Never guess a UOM/pack value or misclassify a location.** Matches
   bakery's equivalent guardrail. T4's location classification follows this:
   two locations (`806`, `402`) that didn't fit the Thai-keyword pattern
   cleanly got an explicit, documented override rather than a guess — see T4
   below.
4. **No partial writes.** Bulk operations validate fully and commit whole, or
   reject and commit nothing.
5. **Every write path gets a log entry** with who, when, and `source`
   (`'ui'` / `'excel-import'` / `'admin-override'`).
6. **Preserve existing behaviour when porting between bakery and packaging**
   — match bakery's existing output format rather than improving it, since
   these two reports get compared side by side. T5's export here is a
   deliberate, documented exception: it carries one extra column
   (`บันทึกโดย`) that bakery lacks, because this repo tracks
   `_meta.updatedBy` and bakery doesn't — matching bakery exactly would mean
   dropping data this repo actually has, which the guardrail isn't asking for.
7. **One commit per ticket, stop for review after each 🔴 P0 ticket** (T1/T2
   already were; nothing left at that priority until T8/T9 unblock).

## Full ticket list

| # | Priority | Scope | Status |
|---|---|---|---|
| T1 | 🔴 P0 | Reject duplicate item codes at load; normalise pack fields | ✅ merged ([#1](https://github.com/justheuristics/packaging-count/pull/1)) |
| T2 | 🔴 P0 | Outlier variance guard + admin exception queue | ✅ merged ([#2](https://github.com/justheuristics/packaging-count/pull/2)) |
| T3 | 🟢 P1 | Remove plaintext password column from bakery admin UI | N/A — **bakery only**, nothing to do here (this repo has no store admin UI at all: `STORES_DATA` is a hardcoded source array, never Firebase-backed) |
| T4 | 🟢 P1 | Location type + date-effective open/closed status (**both apps**) | ✅ merged ([#3](https://github.com/justheuristics/packaging-count/pull/3)) |
| T5 | 🟢 P1 | Export store submission status to Excel (**this repo** — bakery already had this) | ✅ merged ([#4](https://github.com/justheuristics/packaging-count/pull/4)) |
| T6 | 🟢 P1 | Price / priceUom / priceEffectiveFrom on bakery item master | N/A — **bakery only** |
| T7 | 🟢 P1 | Align Thai calendar display (BE) across both apps | ✅ merged ([#5](https://github.com/justheuristics/packaging-count/pull/5)) |
| T8 | 🟡 P2 | Price list bulk import with preview-diff-confirm | ⛔ **blocked** on open question Q3 |
| T9 | 🟡 P2 | Stock-take Excel upload with validation gate | ⛔ **blocked** on open question Q4 |

T1, T2, T4, T5, T7 are done here (T3/T6 are bakery-only). **Stopped here, as
planned, to await Q3/Q4 decisions before T8/T9.**

## What's done

### T1 — duplicate-code validator + pack-field fixes (merged)
`scanItemMasterIssues()` in `ensureItemsSeeded()`/`loadItemsFromDB()`,
surfaced as a persistent `#masterDataAlert` banner + a detailed card on
Overview + a "รหัสซ้ำ" chip on affected entry rows. Fixed the source
`ITEMS_DATA`: coerced 11 string `packRec` values to numbers, de-duplicated
the retired-category `0160220151Y` row (282 items now, was 283).

### T2 — outlier variance guard + admin exception queue (merged)
Mirrors bakery's T2, adapted to this app's category+code keying
(`FRESH`/`TRANSFER`/`NONFRESH` × item code). `OUTLIER_FACTOR = 10`, two-phase
`saveCategory(confirmedFlags)`, admin exception card via
`computeAndPersistOutlierExceptions(month)`.

### T4 — location type classification + date-effective status (merged)
Classifies all 208 `STORES_DATA_RAW` locations into `locationType` (STORE /
DC / FC / DUMMY / VIRTUAL / FROZEN) by name pattern (`คลังสินค้า` / `เอฟซี`
/ `Dummy` / `ร้านค้าเสมือนจริง` / `สยามโฟรเซ่น`): **DC 19, FC 5, DUMMY 1,
FROZEN 9, STORE 172, VIRTUAL 2**. Two locations needed explicit handling
beyond the raw pattern match:

- `806` "Ningbo Beicang (Tmall)" matches no Thai keyword (a virtual
  marketplace listing, not a physical location) — overridden to VIRTUAL via
  `LOCATION_TYPE_OVERRIDE`.
- `402` "คลังสินค้าเอเอฟซี-บางนา" contains both `คลังสินค้า` and (inside
  `เอเอฟซี`) `เอฟซี`; checking `คลังสินค้า` first classifies it DC, matching
  its own name — this **contradicts** the original plan doc's listing of it
  as FC. Trusted the name.

`528`/`801`/`804` classify as STORE by name and stay classified-but-not-
excluded pending open question Q1, behind `EXCLUDE_SIAM_FROZEN_ADJACENT`
(currently `false`) — flip that constant, not the classifier, once Q1
answers.

`isCountableAt(loc, ym)` — same shape and semantics as bakery's — applied to
`loadOverviewStats()` and `computeCategorySums()`/`renderDashboardResult()`.
Verified: admin overview went from reading against the raw 208-location
count to 172 (countable stores only) for August 2026 — the exact "63% vs
true 78%" denominator gap the original plan doc's problem statement
described. No store admin UI exists in this repo (`STORES_DATA` is a
hardcoded array, not Firebase-backed), so there's no delete mechanism to
replace and no status-change UI — this repo's T4 is classification + field
shape + `isCountableAt` only, unlike bakery's UI-heavy half.

### T5 — export store submission status to Excel (merged)
`exportStoreStatusPackaging(month)`, button in the dashboard's
`storeBreakdownSection`, reuses `exportRowsToExcel()` (no new dependency).
Same three sheets as bakery (`ยังไม่บันทึก`/`บันทึกแล้ว`/`ทั้งหมด`), same
column order, plus one extra column (`บันทึกโดย`) bakery doesn't have.
Depends on T4's `isCountableAt()` so the export doesn't list DC/FC/dummy/
virtual/frozen as "missing branches." Verified against real July 2026
production data: 131 submitted + 41 not-submitted = 172, matching
`computeCategorySums()`'s own countable-store count.

### T7 — Thai Buddhist-era calendar (merged)
`thaiMonthLabel()`, `thaiDate()`, `fmtDateTime()` were all Common Era;
applied `+543` to all three, in-app and in exports (`buildExportRow()`
already routes through these, so no separate export-side fix was needed).
Machine-readable date keys (`todayStr()`, `dateRange()`, the
`counts/{YYYY_MM}` key generator) are untouched.

## Departures from the original plan doc

These were verified against actual code/data while implementing T1, T2, T4,
T5, T7:

1. This repo's 208 locations classify as **DC 19, FC 5, DUMMY 1, FROZEN 9,
   STORE 172, VIRTUAL 2** — confirmed at runtime (not just in the static
   source array), which is what caught that the `806` override actually took
   effect (STORE dropped from a raw-pattern 173 to 172 once `806` moved to
   VIRTUAL).
2. Loc `402` classifies DC by its own name, **contradicting** the original
   plan doc's FC listing — trust the name, as flagged in the T4 PR.
3. `528`/`801`/`804` stay classified-but-not-excluded pending Q1, per the
   original plan — the config constant (`EXCLUDE_SIAM_FROZEN_ADJACENT`) is in
   place and defaults to not excluding them.
4. `STORES_DATA` remains a **hardcoded source array**, never Firebase-backed
   — confirmed again while building T4/T5, nothing changed here from what
   T1 already established.

## Before starting T8 or T9

1. **Both are blocked** — do not start either without Q3/Q4 answered. This
   isn't a technical blocker, it's a product decision:
   - **Q1** (separate from Q3/Q4, still open): whether `528`/`801`/`804`
     should actually be excluded from store counts — flip
     `EXCLUDE_SIAM_FROZEN_ADJACENT` once Store Ops decides, no other code
     change needed.
   - **Q3** (gates T8): does uploading a new price list silently restate the
     reported value of every prior month, if price only ever lives on the
     item master? (Bakery's T6 already put price fields on its item master —
     T8 needs a decision on whether historical exports/dashboards should
     snapshot price at count-time instead of always reading current master.)
   - **Q4** (gates T9): store-only upload vs. admin-on-behalf upload.
2. Once unblocked: `git checkout -b ticket-8-price-import main` (or
   `ticket-9-...`), confirm `git log origin/main` shows T1/T2/T4/T5/T7
   merged first.
3. **This repo's `demo/` Firebase rules gap (guardrail 1) is still
   unresolved.** If T8/T9 need a real write-path round-trip test in demo
   mode, get that fixed first, or accept logic-level verification as an
   explicit, discussed tradeoff — not a default.
4. Delete this section (and update the ticket table above) once T8/T9 are
   answered and started, so this file stays a living resume point. Don't
   delete the guardrails/departures sections — they stay relevant.
