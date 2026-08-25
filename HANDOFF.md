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
| T8 | 🟡 P2 | Price list bulk import with preview-diff-confirm | ⛔ **blocked** — see "What still blocks T8/T9" below (Q3 decided and its T11 prerequisite merged; three real blockers remain) |
| T9 | 🟡 P2 | Stock-take Excel upload with validation gate | ⛔ **blocked** — see "What still blocks T8/T9" below (Q4 is decided) |
| T10 | 🔴 P0 | Admin-visible reference band + outlier no-coverage state | ✅ merged |
| T11 | 🟡 P2 | Snapshot price onto the entry at save time (**prerequisite for T8**) | ✅ merged — see Q3 below |

T1, T2, T4, T5, T7, T10 and T11 are done here (T3/T6 are bakery-only). **This repo
led T11** — bakery's is still open and may follow this shape, or diverge, since its
price model differs (T6 gave it `price`/`priceUom`/`priceEffectiveFrom` +
`estCostOf()`). Each repo keeps its own `HANDOFF.md`; this section does not speak
for bakery.

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

`528`/`801`/`804` classify as STORE by name. At T4 time this stayed
classified-but-not-excluded pending open question Q1, behind
`EXCLUDE_SIAM_FROZEN_ADJACENT`. **Superseded 25 Aug 2026** — see "Follow-up
call, 25 Aug 2026" below: 801/804 are now excluded via `EXCLUDED_STORE_CODES`,
528 stays counted, and the flag no longer exists.

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

### T11 — snapshot price at save time (merged)

Q3, decided 25 Aug 2026: history freezes at count-time prices. `itemUnitCost(item)`
always read `item.price` **live** from the item master, so an admin editing one
price on the item screen silently restated every prior month's reported value —
including the T10 reference-band comparison. This repo led T11; bakery's is still
open (see the ticket table above).

**The snapshot shape.** Two fields stamped onto a count record at save time:
`price_at_count` and `pack_at_count`. Both are stamped, not just price — the app's
own math (`amount = (qty·pack + sub) · (price/pack)`) means a `packCount` change
alone would still move a "frozen" amount if only price were captured. Three new
helpers sit beside `itemUnitCost()`: `hasPriceSnapshot(rec)` (true only when both
fields are finite and `pack_at_count > 0` — a malformed stamp falls back rather
than dividing by zero or producing `NaN`), `effectivePackCount(item, rec)`, and
`effectiveUnitCost(item, rec)`. `recordTotal()` and `recordAmount()` — the two
chokepoints every money and quantity figure in the app already routes through —
switch to the `effective*` helpers, so the dashboard, the reference band, history,
the exception queue, the admin data table and both Excel exports all pick this up
without touching a single call site. `itemUnitCost(item)` itself is unchanged; it's
what `effectiveUnitCost` falls back to.

**No backfill — decided, not an oversight.** Every record that existed before this
ticket has no snapshot and keeps reading the live master price, same as before.
Fabricating a historical price the app never recorded is the same mistake as
fabricating a reference band, which T10 explicitly refused to do. Verified against
real July 2026 production data: **all 4,042 rows report as still floating on live
price** (0 snapshotted, since none predates this ticket) — proof the read side is a
true no-op for existing data, and the FRESH total (฿291,122,691.35), the 131/172
submitted count, the band summary (89/42/0), and stores 153/159's out-of-band
figures were all bit-for-bit unchanged after this landed.

**The fallback is visible everywhere money is shown**, mirroring T10's "missing ≠
passed" rule: a muted `ราคาปัจจุบัน — ยังไม่ได้ตรึง` pill on un-stamped rows in the
admin data table; a `ราคาฐาน` column in the Excel export stating the basis
(`ราคา ณ วันที่นับ` vs. `ราคาปัจจุบัน — ยังไม่ได้ตรึง`) per row; a count of how many
of the month's rows are still un-stamped on the admin dashboard
(`livePriceNoteHtml()`, counted inside `computeCategorySums()`'s existing loop —
no extra Firebase read); and a warning in `saveItem()`'s modal and confirm toast,
on the screen where a live-price restatement actually originates, stating plainly
that already-stamped months are unaffected and un-stamped months will move.

**Fixed a pre-existing bug found while building this.** `saveEditedRecord()` (the
admin's inline data-edit) rebuilt the record from scratch — `{ qty, subunit_qty,
counted_at }` — which already silently dropped T2's `confirmedBy`/`confirmedAt`/
`flagReason` on every admin edit, wiping the outlier-confirmation audit trail. It
would have dropped the price snapshot the same way. Now spreads the existing
record and overwrites only the edited fields — carrying the snapshot **and** the
T2 fields forward, and **never re-stamping**: an admin correcting a past month's
quantity must not silently re-price that month at today's rate. A legacy row with
no snapshot stays un-snapshotted after an edit.

**Consequences worth knowing, not fixed by this ticket:**
- `PRIOR_MONTH_TOTALS`'s qty-ratio comparison (T2) now compares each month on its
  own pack basis once both months are snapshotted — previously a `packCount`
  change between months was invisible, hidden behind a coincidentally-unmoved
  ratio for zero-`subunit_qty` rows.
- A deleted master item's historical amounts no longer silently zero out.
  `placeholderItem()` returns `price: 0`; an un-snapshotted record for a deleted
  item already read ฿0 for its history, a snapshotted one now survives deletion.
- The first month with a mix of stamped and un-stamped rows blends both price
  bases into that month's `stats/{ym}/itemMedian`. Self-corrects after one month;
  recorded, not fixed.

**Departure — write-path verification without a production write.** T11 is the
first write-path ticket since T2, and guardrail 1's `demo/` gap (blocker #3 below)
is still unfixed, so a real round-trip there is impossible. Per that guardrail,
verified logic-level instead: stubbed `dbUpdate` in a live session to capture its
payload instead of sending it, drove a real `saveCategory()` as a logged-in store,
and confirmed the captured record carried `price_at_count`/`pack_at_count` equal to
the live master at save time, `_meta` present, and no other paths touched — then
restored the stub. No production write; `DB_ROOT` stayed `''` throughout.

## Departures from the original plan doc

These were verified against actual code/data while implementing T1, T2, T4, T5,
T7, T10 and T11 (items 1–4 predate T10/T11; items 5–8 are their own):

1. This repo's 208 locations classify as **DC 19, FC 5, DUMMY 1, FROZEN 9,
   STORE 172, VIRTUAL 2** — confirmed at runtime (not just in the static
   source array), which is what caught that the `806` override actually took
   effect (STORE dropped from a raw-pattern 173 to 172 once `806` moved to
   VIRTUAL).
2. Loc `402` classifies DC by its own name, **contradicting** the original
   plan doc's FC listing — trust the name, as flagged in the T4 PR.
3. `528`/`801`/`804` stayed classified-but-not-excluded pending Q1 at T4 time,
   per the original plan, behind the config constant
   (`EXCLUDE_SIAM_FROZEN_ADJACENT`, defaulting to not excluding them).
   **Superseded 25 Aug 2026** — see "Follow-up call, 25 Aug 2026" below:
   801/804 now excluded, 528 retained, flag replaced by `EXCLUDED_STORE_CODES`.
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
7. **T11 departure — write-path verification used a logic-level stub, not a real
   round-trip.** Guardrail 1's `demo/` gap is still open; see T11's "What's done"
   entry above for exactly what was stubbed and why.
8. **T11 departure — fixed a bug outside T11's stated scope.**
   `saveEditedRecord()` rebuilding the record instead of spreading it predates
   T11 and already dropped T2's confirmation fields on every admin edit; T11
   would have inherited that same bug for its own snapshot fields, so the fix
   went in as part of this ticket rather than being filed separately.

## Decisions closed on 25 Aug 2026 — do not re-raise these

Q1, Q3 and Q4 were open questions that gated T8/T9. **All three were decided by
the project owner on 25 Aug 2026.** They are recorded rather than deleted so the
reasoning survives. A future session should not stop on any of them, and should
not reopen them without a new instruction from the project owner.

- **Q1 — DECIDED (25 Aug 2026, morning): keep 528 / 801 / 804 counted.**
  `EXCLUDE_SIAM_FROZEN_ADJACENT` stayed `false`. **Superseded the same day** on
  a follow-up call — see "Follow-up call, 25 Aug 2026" below. Recorded here
  rather than deleted so the reasoning trail (and the fact that this was a
  same-day reversal, not new information arriving later) survives.
- **Q3 — DECIDED: snapshot the price onto the entry at save time.** History
  freezes at count-time prices. **Implemented by T11** (see its "What's done"
  entry above for the field shape and the no-backfill rule). Records saved from
  25 Aug 2026 onward carry `price_at_count`/`pack_at_count` and are immune to
  later price edits; every record saved before T11 has no snapshot and keeps
  reading the live master price, marked visibly wherever money is shown
  (`ราคาปัจจุบัน — ยังไม่ได้ตรึง`) rather than silently. There was never a
  T6-style migration needed here — `item.price` has been on the embedded item
  master from the start; T11 was purely the snapshot-at-save logic.
- **Q4 — DECIDED: store-only upload.** No admin-on-behalf path in T9, and
  therefore no `submittedBy` case-branching to build. **Known open
  consequence, recorded honestly:** the de facto recovery path when a store
  cannot upload is an admin logging in as that store, which the audit trail
  will attribute to the store, not to the admin. That is accepted for now. It
  is a known limitation of the decision, not an oversight in it.

## Follow-up call, 25 Aug 2026 — Q1 revisited + outlier threshold lowered

A second call the same day (25 Aug 2026) revisited Q1 and made one further
change. Both are packaging-only.

**Q1 revisited — 801/804 now excluded, 528 retained.** The morning's "keep all
three counted" decision (see Q1 above) didn't survive the day. 801 and 804 are
now excluded from counting; 528 stays in. The prior code couldn't express that
split — one boolean (`EXCLUDE_SIAM_FROZEN_ADJACENT`) gated one three-code array,
which only works while all three move together. Since they no longer do, the
flag is gone: `EXCLUDED_STORE_CODES = ['801', '804']`, tested directly in
`isCountableAt()`. Same effect as T4's DC/FC/FROZEN exclusion — not counted
toward completion, dashboard sums or the store-status export — 801/804 remain
visible in admin dropdowns/logs and can still log in and save, exactly as
DC/FC/FROZEN locations do today.

`bakery-count` has the same three store codes (528, 801, 804 — แจ้งวัฒนะ528,
แจ้งวัฒนะ801, บางบอน804), which the naming suggests are the same physical
locations as packaging's. **This exclusion does not apply to bakery.** Scoped
to packaging only, per this call. Bakery has no equivalent exclusion mechanism
today (all locations are `locationType: 'STORE'`; see bakery's own T4 comment
in its `app.js`), so matching this there would need its own decision and its
own ticket, not a constant flip.

**`OUTLIER_FACTOR` lowered 10× → 2×**, per project owner decision (call, 25 Aug
2026), to catch anomalies earlier. Known trade-off, flagged at the time and not
yet resolved: at 2× the guard will fire on ordinary variance — seasonal swings,
delivery timing around month-end — far more often than at 10×, which risks
people learning to tick through the confirmation and dulling the control on the
rows where it matters. Watch for this once live; if confirm-rate climbs,
consider a two-tier version (warn at 2×, hard-confirm at a higher threshold)
rather than reverting outright. This paragraph is the record, not a
recommendation to act on yet — the threshold stays at 2× as decided. Same
change made in `bakery-count` the same day, in that repo's own session — see
that repo's `HANDOFF.md` for its mirror of this note.

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
   trade-off — T11 already took this trade-off once (see its departure entry
   above); T8/T9 will need the same call made explicitly again, since bulk
   writes are a larger blast radius than T11's single-record save path.
   **Owner: project owner.**

T11 (Q3's prerequisite for T8) is merged, so that sequencing blocker is cleared —
only the three above remain.

When starting: `git checkout -b ticket-8-price-import main` (or `ticket-9-…`),
and confirm `git log origin/main` shows T1–T7, T10 and T11 merged first. Update
this section as blockers clear, so the file stays a living resume point. Don't
delete the guardrails/departures sections — they stay relevant.
