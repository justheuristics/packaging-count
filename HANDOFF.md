# Handoff — Ticket 2: Variance guard + admin exception queue

Working from `CLAUDE_CODE_ACTION_PLAN_Counting_Apps.md`. Ticket 1 (duplicate-code
validator + pack-field type fixes) is merged to `main`. This branch
(`ticket-2-outlier-guard`) adds the outlier variance guard and the admin
exception queue, mirroring the equivalent work already fully verified in the
sibling `bakery-count` repo. **Read this file before touching this branch** —
it's the state at handoff, not a design doc; delete it once T2 is merged.

## What's here

All in `index.html`, in the section headed `OUTLIER GUARD (T2)` plus the two
call sites it plugs into. This app has no module system, so the logic is a
direct duplicate of bakery's — not shared code — adapted to this app's
category+code keying (`FRESH`/`TRANSFER`/`NONFRESH` × item code) instead of
bakery's flat code-only keying:

- `OUTLIER_FACTOR = 10` — next to `DB_ROOT`.
- `OUTLIER_CONTEXT_PROMISE`, `prevMonthKey()`, `loadOutlierContext()`,
  `evalOutlier(...)`, `computeOutlierFlag(cat, code, rec)` — same two-ratio
  logic as bakery (qty vs. own prior month, symmetric; value vs. network
  median, one-directional overstatement only). `loadOutlierContext()` runs
  fire-and-forget from `loadMonthData()` (doesn't block the instant
  embedded-data table render); `saveCategory()` awaits
  `OUTLIER_CONTEXT_PROMISE` before computing flags so it never races the
  background load.
- `showOutlierConfirmModal()` / `saveCategory(confirmedFlags)` — same
  two-phase save as bakery's `doSaveEntry`. First call computes flags from
  the lines that actually changed; if any exist, shows a modal and returns.
  Confirm re-calls `saveCategory(flagged)`, which stamps
  `confirmedBy`/`confirmedAt`/`flagReason` onto each flagged record in
  `updates` before `dbUpdate`.
- `computeAndPersistOutlierExceptions(month)` / `exceptionQueueCardHtml(...)`
  — admin dashboard card, wired into `loadDashboard()`/`renderDashboardResult()`
  via module-level `DASH_EXCEPTIONS`. Fetches `counts/{key}` and the prior
  month's `counts/{prevKey}` directly (separate fetch from
  `computeCategorySums`, since that function doesn't return the raw per-store
  data and doesn't fetch the prior month at all). Persists
  `stats/{key}/itemMedian` and `.../computedAt` every time an admin opens the
  dashboard with "ทุกสาขา" (ALL stores) selected.

## Estimated-cost labeling — this app had none before this ticket

Packaging had **zero** "ต้นทุนประมาณการ — ไม่ใช่มูลค่าอย่างเป็นทางการ"-style
disclaimers anywhere before this pass (confirmed by grepping the pre-T2 file
for "ประมาณการ" — zero hits). Bakery already had extensive labeling from prior
work. This ticket swept ~9 in-app spots (entry table header/footer, reference
band ×2, entry totals line, history table, admin overview stat card, admin
data table, dashboard hero, the new exception card) and renamed the Excel
export column key `'มูลค่า (บาท)'` → `'มูลค่า (บาท) — ต้นทุนประมาณการ
ไม่ใช่มูลค่าอย่างเป็นทางการ'` everywhere it's used as a header (6 occurrences,
in `buildExportRow` and `runAdminExport`'s summary logic — confirmed via
`replace_all` that the two unrelated bare `<th>` table-header occurrences of
the same substring were *not* touched, since those aren't quoted object-key
literals).

## Verification status — read this before assuming it's fully proven

**Verified via direct function calls and via the real `saveCategory()` code
path with manually-seeded context** (not a full Firebase round trip — see
why below):
- `computeOutlierFlag()` called with synthetic before/after values (5→60,
  12x) returns the correct flagged result.
- Set a real qty input to 60, manually seeded
  `PRIOR_MONTH_TOTALS['FRESH']['<code>'] = 30`, called the real
  `saveCategory()` — the modal appeared with the correct code, description,
  and ratio text. Ticked the checkbox, called `confirmOutliersAndSave()`,
  it completed without error.
- T1's validator/banner still render correctly (0 issues) after reload —
  confirms this ticket's edits didn't regress T1.

**NOT verified: a real two-month save → reload → flag cycle against
Firebase.** Packaging's `demo/` Firebase root has **no working security
rules** — every read/write under `demo/` returns `PERMISSION_DENIED` and
falls back to localStorage (confirmed via console: `demo/items`,
`demo/settings/months`, `demo/counts/...`, `demo/stats/...`,
`demo/presence/...` all denied). This is **pre-existing behavior**, not
something T1/T2 introduced — there's already a `firebaseRulesCard` warning
UI in this codebase for exactly this scenario (see `setMonthStatusDirect`).
The localStorage fallback only caches **individual leaf paths**, not whole
subtrees, so `loadOutlierContext()`'s parent-path read
(`counts/{prevKey}/{storeCode}`) can never be satisfied from the fallback
cache even after a successful leaf-level fallback write. That's why the
smoke test above had to manually seed `PRIOR_MONTH_TOTALS` in memory rather
than relying on a real save/reload cycle.

**Decision made**: rather than fixing packaging's Firebase rules first, the
above logic-level + partial-live verification was accepted as sufficient,
since this code is a direct structural port of bakery's already
fully-verified logic. If you want the stronger guarantee, either get
packaging's `demo/` root opened up the way bakery's already is, or test
against a real (non-demo) path with extreme care and full awareness that's
production data — do **not** do the latter without explicit sign-off.

**Also noted, not fixed**: right after a real `saveCategory()` write, the
in-memory `CURRENT_DATA`/`ENTRY_SAVED_BASELINE` mirror does not carry the
newly-stamped `confirmedBy`/`confirmedAt`/`flagReason` (only the `dbUpdate`
payload that reaches Firebase does). This mirrors a pre-existing gap the
original code already had for `counted_at` — not a regression, just
inherited. Only matters if something later reads those fields from
in-memory state without a reload.

**Never live-verified**: the admin exception card actually rendering with
real flagged data in packaging's dashboard UI (blocked by the same
demo-Firebase-rules gap above — never got a full real save cycle far enough
to reach that screen with genuine data). The function was read carefully
against bakery's already-verified equivalent and is structurally identical;
treat this as logic-reviewed, not click-tested.

## Before committing / resuming

1. `DB_ROOT` is `''` (production) as of this handoff.
2. Syntax check: this file has no separate JS file — extract the inline
   `<script>` block and run `node --check` on it (see chat history for the
   exact Python extraction command; needed because of git-bash vs. native
   Windows Python path quirks with `cygpath -w`).
3. Guardrail 8: this is a 🔴 P0 ticket — stop for review after committing,
   before starting T3.
4. Guardrail 7 (bakery/packaging output-format parity): the outlier
   confirmation modal's copy, the exception card's column layout, and the
   estimated-cost disclaimer wording were all written to match bakery's
   equivalents as closely as this app's category-scoped data model allows.

## Not done yet (repo-agnostic, tracked in the shared plan)

T3–T7 not started. Full 7-ticket plan and cross-repo status lives in the
plan file used by the Claude Code session that did this work — ask the user
for it if you need the broader picture; this file only covers what's on this
branch, in this repo.
