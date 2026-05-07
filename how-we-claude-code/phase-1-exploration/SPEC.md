# Bill-Splitting App for Couples — Spec

**Date:** 2026-05-07
**Phase:** 1 — Exploration (output)
**Audience:** Couples sharing finances continuously.
**Working name:** TBD (placeholder: "Diário do Casal").

---

## 1. Product Summary

A no-account, link-based ledger that splits a couple's shared expenses
proportionally to each partner's monthly income, maintains a continuous
running balance, and lets either partner record expenses, transfers, and
recurring bills from any device.

**Primary use case:** two people living together (or sharing finances)
register shared expenses as they happen. The app figures out how much each
owes the other based on their incomes for that month, accumulates the
running balance over time, and lets them settle up by recording manual
transfers (Pix, transfer, cash — actual payment happens outside the app).

**What it explicitly is not:**

- Not a budgeting tool (no categories, envelopes, goals).
- Not an audit/log tool (no per-entry edit history).
- Not a payment processor (no Pix integration, no QR generation).
- Not a personal finance tracker (only joint expenses go in).

---

## 2. Decisions Locked In (from Phase 1 interview)

| Question | Decision |
|---|---|
| Audience | Couples / joint finances. |
| Core mechanic | Settle-up proportional to income. |
| Money flow | Each partner pays with their own card/account; app reconciles. |
| Income model | Variable month-to-month. Either partner enters their own income for the current month at any time. Late-month income input retroactively recomputes proportions for that month's expenses. |
| Income privacy | Both partners see each other's income value (full transparency). |
| Authentication | No accounts. The ledger is identified by a random URL slug. Anyone with the link has full access. |
| Onboarding | Creator generates the ledger, types both members' names, copies the link, sends it to the partner. Both partners share identical access; the link itself does not distinguish "who you are." |
| Cycle / settle | Continuous running balance. No month-end closing event. Expenses and transfers are timestamped events; the balance is always derived. |
| Scope of expenses | Joint expenses only. Personal expenses are not tracked here. |
| Settle-up | Manual transfer events ("I sent R$ 200 by Pix") recorded explicitly. The app shows the current balance but does not handle payment itself. |
| Recurring expenses | Yes. Auto-materialized from rules on a configured day-of-month. |
| Missing income for current month | Fallback to the partner's previous month's income; if neither partner has any history, fallback to 50/50. New income input recalculates that month's expenses. |
| Stack | Vite + React + TypeScript + Bun, aligned with phase 3 of the workshop. |

---

## 3. Domain Model

### Entities

```
Ledger
  slug: string (16+ char base62, URL-safe, unguessable)
  createdAt: timestamp

Member  (exactly 2 per ledger)
  id: uuid
  ledgerSlug: string
  name: string
  position: "A" | "B"   // stable, not displayed; used to seed deterministic ordering

IncomeEntry
  memberId: uuid
  monthKey: string      // "YYYY-MM"
  amountCents: integer  // > 0
  updatedAt: timestamp
  // Unique on (memberId, monthKey). Latest write wins.

Expense
  id: uuid (client-generated)
  ledgerSlug: string
  paidByMemberId: uuid
  amountCents: integer  // > 0
  description: string
  occurredAt: timestamp
  createdAt: timestamp
  recurrenceRuleId: uuid | null
  deletedAt: timestamp | null   // soft-delete (only for recurrence-materialized rows; see §6)

Transfer
  id: uuid (client-generated)
  ledgerSlug: string
  fromMemberId: uuid
  toMemberId: uuid
  amountCents: integer  // > 0
  occurredAt: timestamp
  note: string | null
  createdAt: timestamp

RecurrenceRule
  id: uuid
  ledgerSlug: string
  paidByMemberId: uuid
  amountCents: integer
  description: string
  dayOfMonth: integer (1..31)   // clamped to last day of short months
  startsAt: monthKey            // first month to materialize
  endsAt: monthKey | null       // exclusive; null = open-ended
  pausedAt: timestamp | null
```

All amounts stored as integer cents to avoid float drift. Currency is BRL only
in the MVP (no multi-currency support).

### Computed Functions (pure, derived from events)

```
proportionFor(monthKey) -> { a: number, b: number, source: enum, fallbackFromMonth?: monthKey }
  Resolution order:
    1. Both members have an IncomeEntry for monthKey   -> source = "set"
    2. Only one member has an entry for monthKey       -> for the member who is
                                                          missing in monthKey,
                                                          use that same member's
                                                          most recent prior
                                                          IncomeEntry (any
                                                          monthKey < current);
                                                          source = "partialFallback",
                                                          fallbackFromMonth = that prior monthKey
    3. Neither has an entry for monthKey               -> use the most recent
                                                          prior monthKey where
                                                          BOTH members had
                                                          entries;
                                                          source = "fullFallback",
                                                          fallbackFromMonth = that prior monthKey
    4. No history anywhere                             -> { a: 50, b: 50 },
                                                          source = "default5050"
  Returned percentages always sum to 100, rounded to 2 decimals;
  the underlying ratio is stored as a fraction for cent-exact math.

balance(memberId) -> integer cents
  Sum over all (non-deleted) expenses E in the ledger:
    if E.paidBy == memberId:    contributes  +(otherMemberShare(E))
    else:                       contributes  -(memberShare(E))
  where memberShare(E) = round(E.amount * proportionFor(monthOf(E))[member] / 100)
  Plus over all transfers T:
    if T.from == memberId:      contributes  -T.amount
    if T.to == memberId:        contributes  +T.amount
  Always: balance(A) + balance(B) === 0.
```

### Invariants

- `balance(A) + balance(B) === 0` at all times.
- `proportionFor(m).a + proportionFor(m).b === 100` for every monthKey `m`.
- Updating an `IncomeEntry` for `monthKey` recomputes balances for every
  expense in `monthKey` (and only in `monthKey`).
- Renaming a `Member` does not change any balance.
- Editing/deleting an `Expense` or `Transfer` recomputes the balance.
- The set of `(recurrenceRuleId, monthKey)` for non-deleted expenses is a
  subset of all valid `(rule, month)` pairs implied by the rule's
  `startsAt`/`endsAt` and `pausedAt`.

---

## 4. UI / UX

### Screens

1. **Onboarding** (one-time, for a fresh ledger)
   - Step 1: name member A.
   - Step 2: name member B.
   - Step 3: "share this link" — display the new slug URL with a copy
     button and a one-line warning that anyone with the link sees
     everything. Continue to main.

2. **Main — Diário do mês corrente** (default)
   - **Top bar:** month selector `◀ Maio 2026 ▶`. Disabled forward at
     current month.
   - **Balance header:** "Você deve R$ 240 ao Diego" (or symmetric).
     Beside it: proportion pill ("55 / 45") with a tooltip explaining
     the source (set / partial fallback / full fallback / default).
   - **Income block:** two cards side by side, one per member, showing
     the current month's income. Inline edit. Visual indicator when the
     value is a fallback (italic + "vindo de abril/2026").
   - **Timeline:** chronological list of expenses + transfers for the
     selected month. Each row: description, amount, who paid (or
     direction for transfers), per-member share for expenses, recurrence
     icon if materialized from a rule. Inline edit; delete with a 5s
     undo toast.
   - **Floating "+":** opens the entry form (mode toggle: expense /
     transfer).
   - **Month breakdown (footer or drawer):** total spent, your share,
     your contribution, delta against the previous month
     ("vocês gastaram R$ 320 a mais que abril").

3. **Settings** (menu)
   - Edit member names.
   - Recurrence rules: list, create, pause/unpause, delete.
   - Copy ledger link again.
   - Export ledger as JSON (the only escape hatch if the link is lost).

### Components (verifiable units)

- `BalanceHeader` — balance + proportion + fallback indicator for the
  selected month.
- `IncomeCard` — one per member; inline edit; states `idle | editing | saving | error`.
- `MonthTimeline` — sorted list of `EntryRow` for the selected month.
- `EntryRow` — polymorphic over `expense | transfer`.
- `MonthSelector` — controls bounded by oldest entry .. current month.
- `MonthBreakdown` — totals + delta-vs-previous insight.
- `EntryForm` — modal for create/edit; mode toggles between expense and transfer.
- `RecurrenceManager` — settings-screen unit.
- `OnboardingFlow` — three-step wizard for a fresh ledger.

### UI States

- `ledger: Ledger | null` — null shows "ledger not found" screen with CTA
  to start a new one.
- `selectedMonthKey: string` — defaults to current month.
- `editingEntryId: string | null` — controls inline edit on `EntryRow`.
- `pendingPatches: Set<patchId>` — used for optimistic UI; reconciled on PATCH ack.

### Error Handling on the Surface

| Error | UI behavior |
|---|---|
| Slug not found (`GET` 404) | Full-page state with CTA "Create new ledger". |
| PATCH network error | Toast with retry; optimistic state preserved until success or manual undo. |
| PATCH stale (etag mismatch) | Toast "your partner just made a change", refetch, replay local optimistic ops if still valid. |
| Form validation (amount ≤ 0, blank description) | Block submit; field-level message. |
| Onboarding name blank | Block "Continue"; field-level message. |

---

## 5. Persistence & Sync

### Slug

- 16+ chars, base62, unguessable. Single secret. Whoever has the link has
  full read+write. No additional auth.
- Spec note: the slug is the credential. There is **no recovery** for a
  lost slug; the JSON export is the only mitigation.

### REST API (minimal)

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/api/ledger` | Create. Body `{ memberAName, memberBName }`. Returns `{ slug }`. |
| `GET` | `/api/ledger/:slug` | Returns the full document. 404 if absent. |
| `PATCH` | `/api/ledger/:slug` | Apply one mutation. Body `{ op, payload, clientId, clientSeq, baseEtag }`. Returns `{ document, etag }`. |
| `GET` | `/api/ledger/:slug/stream` | Server-Sent Events stream of mutations. Optional in MVP; polling is the fallback. |
| `GET` | `/api/ledger/:slug?format=json` | Export — same as plain GET, formatted for human-savable backup. |

`op` is an enum: `addExpense | updateExpense | deleteExpense | addTransfer |
updateTransfer | deleteTransfer | setIncome | renameMember | addRecurrence |
updateRecurrence | pauseRecurrence | deleteRecurrence`. Each carries a typed
payload validated server-side.

### Storage

- SQLite (`bun:sqlite` or `better-sqlite3`).
- Normalized tables matching §3 entities.
- Unique constraint on `(recurrence_rule_id, month_key) WHERE deleted_at IS NULL`
  for recurrence idempotency.
- Single-process backend; no replication needed for MVP.

### Concurrency

- Two-writer system, low contention. Last-write-wins per entity.
- Optimistic UI: client applies locally, sends PATCH, reconciles on ack.
- ETag on the document; PATCH includes `baseEtag`. On mismatch, client
  refetches and merges by replaying any still-valid local ops.
- Client-generated UUIDs for `Expense`/`Transfer` so simultaneous creates
  don't collide.

### Sync

- **MVP default:** poll `GET /api/ledger/:slug` every 10 s when the tab is
  visible; pause when hidden.
- **Optional:** SSE on `/stream` to push deltas. Same client merge logic.

### Pluggable client layer

```ts
interface LedgerClient {
  load(slug: string): Promise<LedgerDocument>
  patch(slug: string, op: PatchOp): Promise<{ document: LedgerDocument; etag: string }>
  subscribe(slug: string, onChange: (doc: LedgerDocument) => void): () => void
}
```

- Production impl: HTTP + polling.
- Verification/test impl: synchronous in-memory store. Used by isolated
  render targets `/verify/:unit/:fixture` to mount components with
  pre-populated state without a backend.

### Recurrence engine — client-side, lazy materialization

On every load (and on month-selection change), the client computes:

```
for each active rule in ledger.recurrenceRules:
  for each monthKey in [rule.startsAt .. min(currentMonthKey, rule.endsAt ?? currentMonthKey)]:
    if no expense exists with (rule.id, monthKey) and not deleted:
      enqueue addExpense PATCH for that (rule, month) with the rule's snapshotted amount/description
```

- Materializes only past + current months. Never future months.
- Idempotent via the `(recurrence_rule_id, month_key)` unique constraint;
  duplicate PATCHes are no-ops on the server.
- `dayOfMonth = 31` clamps to the last day of short months
  (e.g., Feb → day 28/29).

---

## 6. Recurrences & Edge Cases

### Recurrence rule edits

- Editing `amountCents` or `description` affects **only future
  materializations**. Already-materialized expenses keep their values.
- Editing `dayOfMonth` affects only future materializations.
- "Pause" stops materialization from the next month onward. The current
  month's already-materialized instance persists.
- Delete rule: prevents future materialization; existing expenses untouched.
- Rationale: a materialized expense is a historical record; partners may
  edit one for a one-off variance (e.g., rent went up this month).

### Editing recurrence-materialized expenses

- Treated as normal expenses. Editing `amount`/`description`/`occurredAt`
  is allowed; the link to `recurrenceRuleId` is preserved.
- Deletion is **soft-delete** (`deletedAt` set), not row removal. The
  unique constraint with `WHERE deleted_at IS NULL` keeps the rule from
  re-materializing into the same month — preventing "I deleted it but it
  came back" surprises.
- Deletion of a non-recurrence expense is a hard delete (no soft state).

### Edge cases

| Case | Behavior |
|---|---|
| Slug does not exist | "Ledger not found. Create new?" full-screen state. |
| Ledger exists but onboarding incomplete | Resume `OnboardingFlow` at the next missing step. |
| Current month has no income from either member (fresh ledger) | Banner "Set this month's income". Expenses use 50/50 default; recompute when income is entered. |
| Only one member set income for current month | Fallback: missing member uses their own previous month's income (most recent available). If they have no history, full-fallback to last month where both were set. If no such month, 50/50. |
| Editing a past income entry | Recomputes balances for every expense in that `monthKey`. No extra confirmation; the expected effect is the whole point. |
| Expense with `amountCents <= 0` | Form blocks; minimum is 1 cent. |
| Transfer that crosses balance sign | Allowed. Balance can swing to the other side ("partner overpaid"). |
| Attempt to delete a member | Not exposed in the UI. Members are fixed at the ledger level. |
| Simultaneous edits by two clients | Last-write-wins per entity. Toast on remote change detection. |
| Concurrent creates (two `addExpense` at the same instant) | Both succeed; client-generated UUIDs avoid collisions. |
| Renaming a member with existing entries | OK. Entries reference `memberId`; no migration. |
| Recurrence rule created mid-month | If `dayOfMonth` is today or earlier in the current month, materializes immediately for the current month. Otherwise, first instance is on the next occurrence. |
| Timezone differences between the two devices | `monthKey` and `occurredAt` are computed in the device's local timezone. Last write wins on conflict. (Acceptable risk — couple is typically co-located.) |
| Browser cache cleared | No data loss; state lives on the server. Only risk is losing the slug. JSON export is the mitigation. |
| Slug leak (shared with a third party) | No technical mitigation in MVP. "Rotate slug" is an explicit v2 candidate. The onboarding share screen warns about the risk. |

---

## 7. Verification & Testing

The app inherits the phase-3 verification architecture:

- `data-verify-*` attributes describe each unit's runtime state at the DOM
  surface.
- Each `VerifiableUnit` registers a Zod props schema, named **fixtures**
  (some marked `probe: true` for adversarial cases), and **invariants**.
- `/verify/:unit/:fixture` mounts a single unit in a known state with no
  app shell.
- `window.__verify` exposes `manifest()`, `current()`, `runAll()` for
  agent and CI consumption.
- Pluggable verifiers: `schema`, `invariants`, `dom-contract`, `a11y`.

### Per-unit DOM contract

| Unit | `data-verify-*` attributes |
|---|---|
| `BalanceHeader` | `unit, month, balance-cents, debtor, proportion-a, proportion-b, proportion-source (set\|partialFallback\|fullFallback\|default5050)` (mirrors §3 `proportionFor.source`) |
| `IncomeCard` | `unit, member-id, month, amount-cents, state (idle\|editing\|saving\|error), source (set\|inherited)` |
| `MonthTimeline` | `unit, month, entry-count, expense-count, transfer-count` |
| `EntryRow` | `unit, kind (expense\|transfer), id, amount-cents, paid-by, recurrence (rule-id?), editing` |
| `MonthBreakdown` | `unit, total-cents, share-a-cents, share-b-cents, paid-a-cents, paid-b-cents, delta-prev-cents` |
| `MonthSelector` | `unit, month, can-prev, can-next` |
| `EntryForm` | `unit, mode (expense\|transfer), state (idle\|submitting\|error), valid` |
| `RecurrenceManager` | `unit, rule-count, active-count` |
| `OnboardingFlow` | `unit, step (names\|share), name-a-set, name-b-set` |

### Fixtures (minimum coverage per unit)

- `BalanceHeader`: `equal-incomes-zero-balance`, `a-paid-more-owes-less`,
  `fallback-from-prev-month`, `fallback-default-5050` (probe),
  `large-balance-r1m` (probe).
- `IncomeCard`: `idle-with-amount`, `editing`, `saving-then-success`,
  `network-error` (probe), `inherited-from-prev-month`.
- `MonthTimeline`: `empty-month`, `mixed-expenses-transfers`,
  `recurrence-only-month`, `single-day-with-50-entries` (probe).
- `EntryRow`: `expense-paid-by-a`, `expense-paid-by-b`, `transfer-a-to-b`,
  `recurrence-instance`, `editing`, `negative-amount-rejected` (probe),
  `zero-amount-rejected` (probe).
- `MonthBreakdown`: `prev-month-larger`, `prev-month-smaller`,
  `no-prev-month`, `current-month-empty`.
- `MonthSelector`: `at-current-month`, `at-first-recorded-month`, `between`.
- `EntryForm`: `valid-expense`, `valid-transfer`, `invalid-amount`,
  `submission-error` (probe).
- `RecurrenceManager`: `no-rules`, `mix-active-paused`,
  `rule-with-day-31-in-feb` (probe).
- `OnboardingFlow`: `step-names-empty`, `step-names-one-filled`,
  `step-share-with-slug`.

### Invariants (per unit)

- `BalanceHeader`: `proportionA + proportionB === 100`. `debtor` is
  consistent with the sign of `balanceCents`. `proportionSource="default5050"`
  implies `proportionA === 50 && proportionB === 50`.
- `IncomeCard`: `state="saving"` ⇒ input disabled. `source="inherited"` ⇒
  fallback indicator visible.
- `MonthTimeline`: `expenseCount + transferCount === entryCount`. Children
  ordered by `occurredAt` desc.
- `EntryRow`: `kind="transfer"` ⇒ no `paid-by`. `kind="expense"` ⇒
  `paid-by` present. Delete button has accessible name.
- `MonthBreakdown`: `shareA + shareB === total` (cent-exact). When no
  prior month exists, the insight node is not rendered.
- `MonthSelector`: `canNext === false` when `month === currentMonthKey`.
- `EntryForm`: `state="submitting"` ⇒ inputs disabled. `valid="false"` ⇒
  submit button disabled.
- `OnboardingFlow`: `step="share"` ⇒ both names set. Slug shown is ≥16 chars.

### Cross-unit invariants

Verified via composed fixtures and `__verify.runAll()`:

- `balance(A) + balance(B) === 0` in every state.
- `Σ MonthBreakdown.shareA across all months === Σ MonthBreakdown.paidA across all months − balance(A)`
  (accounting identity).
- After `addExpense({amount: X, paidBy: A})`: `BalanceHeader.balanceCents`
  changes by exactly `round(X * proportionB / 100)` toward A (in B's debit).
- After `addTransfer({from: A, to: B, amount: Y})`:
  `BalanceHeader.balanceCents` changes by exactly `+Y` in the expected direction.

### New verifier

- `accounting-identity` — runs cross-unit invariants on composed fixtures.
  Catches refactor regressions in the balance computation.

### Headless run

`bun run verify` executes the full unit × fixture × verifier matrix plus
the cross-unit `accounting-identity` checks. CI is green only at 100%.

---

## 8. Out of Scope (MVP)

Explicitly deferred:

- Multi-currency.
- Categories or tagging.
- Personal expenses.
- Per-entry edit history / audit log.
- Proportion-by-renda override per expense (e.g., "this one is 100% mine").
- Pix QR / payment processor integration.
- Slug rotation (recovery from leak).
- Push notifications / email reminders.
- More than two members.
- Export to anything other than JSON.

These are tracked here so a future v2 spec can pick them up without
re-deriving context.

---

## 9. Open Risks

- **Slug as the only credential.** Loss = total data loss; leak =
  unrestricted access. The only mitigation is the user remembering to
  export. If the workshop's phase 3 surfaces real users, slug rotation
  becomes urgent.
- **Last-write-wins** can lose a partner's edit silently in the rare
  case of true concurrent writes. Acceptable for two-person low-contention
  use; revisit if a usage signal shows otherwise.
- **Lazy recurrence materialization** depends on someone opening the app
  at least once a month. For couples using the app actively, fine; if the
  app is left dormant, materialization catches up but balance
  notifications/insights for skipped months are correct only after the
  next open.
