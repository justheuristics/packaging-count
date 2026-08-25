# HANDOFF — Counting Apps action plan (packaging-count)

This repo and its sibling `bakery-count` (`github.com/justheuristics/bakery-count`)
are being worked through a ticket-by-ticket action plan in order, one ticket per
branch, one PR per ticket, merged before the next starts. **Read this whole file
before starting T8, T9 or T11** — it has the guardrails, the full ticket list,
what's already done, which questions are closed, and what's known to depart from
the original plan doc.

If you have the original plan doc (`CLAUDE_CODE_ACTION_PLAN_Counting_Apps.md`),
this file summarizes it plus everything learned while implementing T1–T7 and T10
that the original doc got wrong or didn't anticipate — read this file's departures
section even if you have the original, since some of its stated facts are stale.
The original doc lists 9 tickets; T10 and T11 were added after it was written.

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
7. **One commit per ticket, stop for review after each 🔴 P0 ticket** (T1, T2
   and T10 all did; nothing left at that priority right now).

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
| T8 | 🟡 P2 | Price list bulk import with preview-diff-confirm | ⛔ **blocked** — see "What still blocks T8/T9" below (Q3 is decided; T11 is now its prerequisite) |
| T9 | 🟡 P2 | Stock-take Excel upload with validation gate | ⛔ **blocked** — see "What still blocks T8/T9" below (Q4 is decided) |
| T10 | 🔴 P0 | Admin-visible reference band + outlier no-coverage state | ✅ merged |
| T11 | 🟡 P2 | Snapshot price onto the entry at save time (**prerequisite for T8**) | 📋 not started — design decided 25 Aug 2026, see Q3 below |

T1, T2, T4, T5, T7 and T10 are done here (T3/T6 are bakery-only).

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

### T10 — admin-visible reference band + outlier no-coverage state (merged)

Two controls that were assumed closed but weren't.

**10.1 — the band an admin can actually see.** `renderReferenceBand()` read
`SESSION.locNo` and `computeActiveCategoriesTotal()`, so the store cost band was
visible only to the store itself. The July 2026 anomaly was caught *by this
band* — it is the store-total-vs-band check, a different control from T2's
per-item ratio vs. the network median — and the person who reports the number
upward could not see it. Extracted the three-way classification into a pure
`classifyAgainstBand(total, band) → {status, cls}` and gave it three renderers:
the store panel (unchanged output), the admin dashboard's submitted-stores
list, and `exportStoreStatusPackaging()`. **No band is its own outcome**
(`cls:'none'`, muted), never green — an admin table showing "in range ✅" for a
store with no band is worse than showing nothing. All 131 stores that submitted
July 2026 data have a band. Submitted stores sort red → amber → green →
no-band, so out-of-band stores are reachable without scrolling, and the summary
line above the list keeps the no-band count as its own number. Verified against
real July 2026 production data: store 153 (฿37,302,333 vs. band max
฿233,899) and store 159 (฿221,123,305 vs. band max ฿379,328) both surface as
`สูงกว่าค่าสูงสุด` without drilling into either store — the exact acceptance
case in the ticket.

Per-store totals come from `computeCategorySums()`'s own `submittedStores[].total`
(FRESH+TRANSFER, accumulated in the loop that already runs), not a second pass —
the admin dashboard and `exportStoreStatusPackaging()` both read it, so the two
figures cannot drift apart. Unlike bakery, this repo needed no
`storeMonthTotals()`-style extraction: `computeCategorySums()` was already the
single source for the dashboard and the export before T10.

**10.2 — "could not be checked" is no longer indistinguishable from "passed".**
`evalOutlier()` returned `flagged:false` both when a row passed both ratios and
when neither ratio could be computed at all. Added a third field, `coverage`
(`'full'` / `'partial'` / `'none'`), derived from whether each ratio is
non-null. `flagged` semantics, `OUTLIER_FACTOR` (10), the never-auto-reject
behaviour, and the cross-UOM-safe `recordTotal()` basis are all unchanged.
**No-coverage deliberately does not set `flagged`** — that would put a
confirmation modal in front of every item in its first month, train people to
tick through it, and degrade the control that does work. Instead: a distinct
grey dashed `ยังเทียบไม่ได้` chip on the entry row (visually separate from T1's
red `รหัสซ้ำ` pill), refreshed per-row via the same targeted-DOM-update pattern
`onQtyInput()` already uses, plus an unchecked-row count on the admin exception
card, shown in the "ไม่พบรายการผิดปกติ" branch too — a bare all-clear reads as
full coverage when it may be no coverage. Verified against real July 2026 data:
2,385 of 4,042 rows (59%) had no coverage, 697 partial.

**10.3 — not applicable to this repo.** No `master_uom.json`, no `UOM_LIST`
exist here; bakery's `UOM_VOCAB_REPORT.md` covers the shared UOM-vocabulary
finding. Nothing to produce on this side.

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
5. **T10 departure — `REF_BAND_READY` cannot gate the admin band view.** The
   T10 brief assumed `REF_BAND_READY` "gates whether the file has loaded."
   It doesn't — in this repo it is *store-entry* state, set `false` in
   `loadMonthData()` and `true` only once that store's own month data returns
   from Firebase. An admin never opens the entry page, so using it as written
   would pin the admin band panel in a permanent loading state. Added a
   separate `REF_BAND_LOADED` flag, set once `loadReferenceBand()` finishes
   (success or failure) in `startApp()`, for the admin view specifically —
   `REFERENCE_BAND` and `REF_BAND_READY` themselves are untouched.
6. **T10 departure — 10.3 (UOM vocabulary report) doesn't apply here.** No
   `master_uom.json`, no `UOM_LIST` in this repo. See bakery's
   `UOM_VOCAB_REPORT.md` for the shared finding.

## Decisions closed on 25 Aug 2026 — do not re-raise these

Q1, Q3 and Q4 were open questions that gated T8/T9. **All three were decided by
the project owner on 25 Aug 2026.** They are recorded rather than deleted so the
reasoning survives. A future session should not stop on any of them, and should
not reopen them without a new instruction from the project owner.

- **Q1 — DECIDED: keep 528 / 801 / 804 counted.** `EXCLUDE_SIAM_FROZEN_ADJACENT`
  stays `false`; this is now settled configuration, not a pending question — see
  the comment above the constant in `index.html`.
- **Q3 — DECIDED: snapshot the price onto the entry at save time.** History
  freezes at count-time prices. **This is not implemented by T10 — it is T11,
  and T11 is a prerequisite for T8.** See the T11 row in the ticket table; the
  design is described there and only there, so there is one description of it
  rather than two that can drift apart. **Until T11 lands, `itemUnitCost()`
  still reads the live `item.price` for every month, so historical totals are
  not yet stable** — editing a price on the item master today silently
  restates every prior month's reported value, including the band comparison
  T10 just put on the admin screen. (Unlike bakery, this repo never needed a
  T6-style migration to add price fields — `item.price` has been on the
  embedded item master from the start; T11 here is purely the snapshot-at-save
  logic, no field migration.)
- **Q4 — DECIDED: store-only upload.** No admin-on-behalf path in T9, and
  therefore no `submittedBy` case-branching to build. **Known open
  consequence, recorded honestly:** the de facto recovery path when a store
  cannot upload is an admin logging in as that store, which the audit trail
  will attribute to the store, not to the admin. That is accepted for now. It
  is a known limitation of the decision, not an oversight in it.

## What still blocks T8/T9

These are the real blockers. Each needs an owner to act; none is a coding task.

1. **Five `Non-Cat` item codes** — still colliding on a single Firebase key.
   Needs either real codes or a formal exclusion. **Owner: Buyer / Fresh Food
   packaging (Phat).** Blocks **both T8 and T9**, since both write by item
   code.
2. **Packaging price source** — still PDF-only. Needs a fixed Excel template.
   **Owner: Sornmongkol / CGA, chased via Phat and Sakuntala.** Blocks **T8's
   packaging half only** — bakery's half is source-ready (Code 206, EX VAT).
3. **`demo/` Firebase security rules (guardrail 1)** — unfixed; reads/writes
   under that root return `PERMISSION_DENIED` and fall back to leaf-only
   localStorage. T8 and T9 are both write paths, so this must either be
   fixed, or logic-only verification accepted as an explicit, discussed
   trade-off. **Owner: project owner.** (T10 was unaffected — 10.1 and 10.2
   are read/display changes.)

Plus one sequencing note that is not a blocker but is easy to miss: **T11 must
land before T8**, per the Q3 decision above.

When starting: `git checkout -b ticket-8-price-import main` (or `ticket-9-…` /
`ticket-11-…`), and confirm `git log origin/main` shows T1–T7 and T10 merged
first. Update this section as blockers clear, so the file stays a living resume
point. Don't delete the guardrails/departures sections — they stay relevant.
