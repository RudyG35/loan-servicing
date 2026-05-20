# Loan Servicing — CSR Console

A web-based console for customer service representatives to check loan standing and record payments in real time.

# Loan Servicing — Technical Overview

## Data Model

Two entities, one relationship.

**Loan** — `id`, `borrowerName`, `principal`, `annualRate`, `termMonths`, `monthlyPayment`, `originationDate`, `firstDueDate`, `maturityDate`

**Payment** — `id`, `loanId` (FK → Loan), `amount`, `paidOn`, `dueDate`

```
Loan ──< Payment
  id ←── loanId
```

One **Loan** has zero or more **Payments**. Payments are linked to their loan via `loanId`.

**Notes**:  
No `status` field is stored on Loan. All statuses are derived at query time from payment records vs. the billing schedule, eliminating stale-flag bugs. Payment IDs encode loan and sequence: `P-{loanNumber}-{nn}`.

---

## API Surface

Base URL: `http://127.0.0.1:3001`. All evaluation endpoints accept `?asOf=YYYY-MM-DD` to pin the reference date for deterministic testing.

| Method | Path | Description |
|---|---|---|
| `GET` | `/health` | Liveness → `{ ok: true }` |
| `GET` | `/loans` | All loans with derived `status`, `currentBalance`, `nextDueDate` |
| `GET` | `/loans/:id` | Full loan record + derived fields |
| `GET` | `/loans/:id/payments` | History newest-first, each tagged with `classification` |
| `POST` | `/loans/:id/payments` | Record a payment; returns record, updated evaluation, new balance. Returns `422` if loan is `paid_off`. |
| `GET` | `/loans/:id/evaluation` | Full detail: `status`, `daysPastDue`, `missedCycles`, `onTimeRate12mo`, `summary` |

**POST body:** `{ amount: number, paidOn: "YYYY-MM-DD", dueDate?: "YYYY-MM-DD" }` — `dueDate` defaults to `evaluation.nextDueDate` if omitted. If the amount exceeds what is needed for the target cycle, the remainder is split across consecutive future cycles and returned in `splitPayments[]`.

**Status codes:** `200` OK · `201` Created · `400` validation · `404` not found · `422` business rule violation (paid-off guard).

---

## Evaluation Engine

All business logic lives in `src/evaluation.js` — pure functions, no I/O.

**Schedule.** Monthly cycles from `firstDueDate` through `maturityDate`. `addMonths` clamps to the target month's last day (Jan 31 → Feb 28/29).

**Coverage.** Sum all payments tagged to a cycle's `dueDate`. A cycle is *covered* if the total ≥ `monthlyPayment` (1¢ tolerance), **or** if any payment arrived within the grace window — keeping an on-time partial payment from triggering a missed cycle. Multiple partial payments on the same due date accumulate; once their sum reaches `monthlyPayment` the cycle is fully covered and status returns to `current`.

**Overpayment redistribution.** When payments for a cycle exceed `monthlyPayment`, the excess is carried forward to the next scheduled cycle at evaluation time (pure derivation — no records are modified). Redistribution cascades if the carry-forward itself creates a further overage.

**Grace period: 10 days.** Derived from seed edge cases: L-1007 (8 days late) is within grace; L-1005 (12 days late) is outside.

**Loan status** (evaluated in priority order):

| Status | Condition |
|---|---|
| `delinquent` | 2+ past cycles uncovered and past grace |
| `paid_off` | Today ≥ maturity date and final cycle covered |
| `late` | 1 uncovered cycle past grace, **or** last payment was partial and its due date is > 10 days ago |
| `late_within_grace` | Same as above but due date is ≤ 10 days ago |
| `current` | Everything else |

**Payment classification** (per payment): `on_time` · `late_within_grace` · `late` · `partial` (partial takes priority over timing).

**On-time rate.** Share of evaluated billing cycles with `dueDate ≥ today − 12 months` that were paid in full on time.

- **Window:** evaluated cycles (respects the truncated-history guard) whose `dueDate` falls in the last 12 months — not `paidOn`.
- **Missed cycles ≤ 12 months old** count in the denominator as not on-time.
- **Missed cycles > 12 months old** are excluded from the denominator entirely.
- **Cumulative grading:** payments for a cycle accumulate in `paidOn` order. The cycle is on time when the running total first reaches `monthlyPayment` AND that completing payment's `paidOn` is within `GRACE_DAYS` of the due date. A completing payment past grace leaves the cycle not on-time.
- **Lump-sum override:** if any single `paidOn` date has total payments ≥ 12 × `monthlyPayment`, the on-time rate resets to 100% unconditionally. Routes split a large payment into per-cycle records all sharing the same `paidOn`; detection sums all payments by date.
- Returns `null` when no evaluated cycles fall within the window.

**Truncated-history guard.** Coverage is only evaluated for cycles at or after the earliest payment on file, preventing long-lived loans with partial history (e.g. L-1001) from being falsely flagged as delinquent.

---

## Running

```bash
cd api && npm install && npm start        # API on :3001
cd web && npm install && npm run dev      # UI on :5173

cd api && npm test                        # Vitest unit suite
cd api && node smoke-test.js              # Zero-dependency regression check
```
----

## Persistence

In-memory arrays loaded from `seed-data.json` at startup. `addPayment` appends to the live array only — nothing written to disk, state resets on restart. This keeps infrastructure at zero: `node server.js` is the whole stack and the seed is always the known starting state, making tests deterministic without database setup. A production deployment would swap the arrays for a database write query.

---

## Testing Strategy

**Unit tests** (`evaluation.test.js`, Vitest) — pinned to `TODAY = 2026-05-17`, covering:
- Date helpers: `addMonths` clamping, `daysBetween`, `buildSchedule`
- `classifyPayment`: all branches including the exact 10/11-day grace boundary
- `evaluateLoan`: every labeled seed edge case (L-1001 through L-1008), healthy loans (L-1009–L-1020), and synthetic scenarios (month-end clamping, leap year, paid-off boundary, on-time rate window)
- `computeCurrentBalance`: no-payment identity, monotonic decrease, floor at zero

**Smoke test** (`smoke-test.js`, plain Node) — zero dependencies, runs all 20 seed loans plus three post-payment flows: L-1003 pay-April → `late` → pay-May → `current`; L-1006 cumulative partial top-up → `current` with `nextDueDate` advancing; L-1003 3× overpayment split across consecutive due dates → `current`.

**Not tested:** HTTP routing, CORS, Fastify wiring. Route handlers are thin delegators; integration tests would add noise without meaningful coverage gain at this scale.

---

## Known Risks/Things to cut if out of time

**Timezone handling.** All logic runs at UTC midnight. A client sending a local-time timestamp instead of `YYYY-MM-DD` could attribute a payment to the wrong cycle. The regex validation at the boundary helps but doesn't fully guard against it.

**Sparse on-time rate.** On a new loan with 2 payments, the rate reflects 2 data points — potentially misleading to consumers who assume a full 12-month window.

**Hard-coded grace period.** In production, GRACE_DAYS would be configurable per loan or product; it's a single constant edit away.

**In-memory persistence.** Payments recorded via `POST` survive for the life of the process only and reset on restart. The data module (`src/data.js`) is the single file that would change to add a database.

**Not addressed:** authentication, authorization, pagination, concurrent write safety, persistence across restarts.

**Things to cut if out of time:**    
**On Time Ratio:** if out of time, I would cut the calucalations for on time ratio as CSR console shows when payment is received and due date. Also, app does not let user make future payments if current payment is not completed. 

---

## AI

Produced using **Claude Sonnet 4.6** (Anthropic) in Cowork mode. All content derived from the actual codebase — `seed-data.json`, `evaluation.js`, `routes.js`, `evaluation.test.js`, and `smoke-test.js`. No field names, status codes, rules, or test cases were invented.

One thing that AI got incorrect was the loan statuses vs the payment statuses. When payment is late and total payment amount for due date is equal to monthly payment amount, the loan status was showing late. I updated code to reflect loan status based upon condition and not last payment status, modified test cases, and tested application. Also, onTimeRatio was calculated incorrectly. AI was calculating onTimeRatio as 100% even if payment is partial and remainder is made after due date. Upon using Claude, there was some discrepencies in my ask and Claude's output. After careful consideration and some debugging, I was able to find where the mistake was made and correct it.
