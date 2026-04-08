# WorkLog Settlement System — Technical Assessment


## Q1: Design Choices — The Settlement Contract

### Q1.1 — Scenarios Where Preview and Backend Numbers Diverge

The frontend fetches worklogs once in `useEffect` (on mount) and polls every 30 seconds. The `POST /api/settlements/run/:userId` body carries no worklog IDs or amounts — only the `userId`. The backend independently fetches and recalculates at execution time. Any database change between the initial `fetchWorklogs` call and the settlement POST will cause a divergence. Concrete scenarios:

**1. Adjustment added between fetch and confirm.** An admin loads the review screen at T=0. At T=10s, another admin or an automated system inserts a new adjustment row for one of the displayed worklogs. The frontend still holds the T=0 snapshot. The backend at T=15s calculates with the new adjustment. The remittance total will differ from the preview total by exactly the adjustment amount.

**2. New time segment submitted while the admin is reviewing.** A freelancer submits a new segment to an OPEN worklog (via `POST /api/worklogs/:id/segments`) after the admin's page has loaded. The GET handler did not include this segment. The settlement engine fetches it via `getTimeSegments` and includes it in `calculateWorkLogAmount`. The preview will be understated.

**3. A concurrent settlement run settles some worklogs first.** If `POST /api/settlements/run` (the all-users route) fires concurrently — e.g., a scheduled job — some of the user's worklogs may be marked SETTLED before the per-user request arrives. `getOpenWorkLogsByUser` filters by `status = 'OPEN'`, so those worklogs are excluded from the backend's result set but were present in the frontend's cached state.

**4. A worklog is deleted or manually closed between fetch and confirm.** If an admin removes a worklog or closes it via a direct DB operation, the backend will not include it. The frontend still renders it in the table with a non-zero amount.

**5. Poll fires between two admin actions but not before confirm.** The 30-second poll could fire and update local state mid-review without the admin noticing, causing the displayed total to silently shift. If the poll does not fire before confirm, the admin acts on a stale number regardless.

### Q1.2 — Success Screen Displays Backend Total, Not Preview Total

In `use-settlement.ts`, on a successful `runSettlement` call, the hook sets:

```typescript
setSettlementResult({
  remittanceId: result.remittance.id,
  totalAmount: result.remittance.totalAmount,
});
```

`result.remittance.totalAmount` comes directly from the API response, which is the value calculated and persisted by `createRemittance` in the backend. In `SettlementReview.tsx`, the success branch renders:

```tsx
<strong>${settlementResult.totalAmount.toFixed(2)}</strong>
```

The `previewTotal` variable — computed inline as `worklogs.reduce((sum, wl) => sum + wl.amount, 0)` — is never referenced on the success screen. There is no comparison, no delta display, and no warning. The admin has zero signal that the executed amount differs from what they reviewed. The UX consequence is that an admin who approved $50 can see $350 on the success screen with no indication that the numbers changed, and no audit trail in the UI linking the approval to the execution.

### Q1.3 — No Idempotency Key, Unreliable Networks, and the In-Memory Set

The POST carries no body — no worklog IDs, no snapshot amount, no idempotency key. On a network retry (client-side retry, load balancer timeout-and-retry, or double-tap):

The backend processes the request as a fresh invocation. If the first request completed successfully, the worklogs are already SETTLED in the database, so `getOpenWorkLogsByUser` returns nothing, and the second invocation returns `{ message: "No open worklogs found for this user." }`. This is a correct outcome in the happy path, but only because the DB status update acts as a guard.

If the first request succeeded in settling all worklogs but `createRemittance` failed (throwing before writing), a retry would find no OPEN worklogs (they are already SETTLED) and would again return the empty message — now incorrectly, since no remittance was ever created. Bob's worklogs are permanently SETTLED with no corresponding payment record.

The `settledWorkLogIds` Set is process-scoped and ephemeral. It helps only within the same server process lifetime. On retry to a different server instance (load balancer round-robin), the set is empty and provides no protection. On a server restart between the original request and a retry, it provides no protection. The Set does nothing for network-level retries in a distributed or restarted environment — it is a false sense of safety for the specific failure mode most likely to occur in production.

### Q1.4 — Robust Settlement Contract

**Frontend should send:** A settlement request body containing the explicit list of worklog IDs the admin reviewed, their client-side computed amounts, the preview total, and a client-generated idempotency key (UUID):

```typescript
{
  idempotencyKey: string,
  worklogIds: string[],
  expectedTotal: number
}
```

**Backend should validate before executing:**
1. Look up the idempotency key in a `settlement_requests` table. If it already exists with a COMPLETED status, return the original remittance without re-executing.
2. Verify that every worklog ID in the request is still OPEN.
3. Recalculate amounts server-side and verify the computed total is within an acceptable tolerance of `expectedTotal`. If it diverges beyond the threshold (e.g., $0.01), reject with HTTP 409 Conflict, returning the current amounts so the frontend can refresh and present updated numbers to the admin.
4. Execute the settlement within a single database transaction: update all worklog statuses, create the remittance, and create all line items atomically. On any failure, the transaction rolls back entirely — no partial settlements exist.

**Retry handling:** The idempotency key persisted before settlement begins allows any retry to return the same result. The client must store the key for the duration of the operation and resend it on retry.

---

## Q2: Trace the State

### Q2.1 — Component State After ADJ-004 Is Inserted (Before Poll)

The `useSettlement` hook fetched worklogs once on mount. The poll interval has not fired. React state is immutable until `setWorklogs` is called — no external mutation affects it. The component still holds the original snapshot:

- WL-003: amount = −$250.00 (segments $250 + ADJ-002 −$500)
- WL-004: amount = $300.00

`previewTotal` = −250 + 300 = **$50.00**. The Confirm button still reads **"Confirm Settlement — $50.00"**.

### Q2.2 — Full Trace Through to Remittance

`confirmSettlement()` calls `runSettlement("USR-002")` which POSTs to `/api/settlements/run/USR-002`. The body is empty.

`runSettlementForUser("USR-002")` calls `db.getOpenWorkLogsByUser("USR-002")`, returning WL-003 and WL-004 (both OPEN).

For WL-003, `db.getAdjustments("WL-003")` now returns **ADJ-002 (−$500) and ADJ-004 (+$300)**. `calculateWorkLogAmount` computes:

```
segments: (2×50) + (3×50) = 250
adjustments: −500 + 300 = −200
total: 250 + (−200) = 50
```

WL-003 amount = **$50.00**

For WL-004: segments (5×60) = 300, no adjustments. Amount = **$300.00**

Total remittance = 50 + 300 = **$350.00**. `createRemittance("USR-002", 350)` is called.

### Q2.3 — Admin Sees $350.00 on the Success Screen

The success screen renders `settlementResult.totalAmount` which is `result.remittance.totalAmount` = **$350.00**. The admin reviewed and approved **$50.00**. There is no warning, no diff, no indication of the change.

The discrepancy could have been caught or prevented at:

- **Backend (`settlement.ts`):** If the request included `expectedTotal: 50.00`, the backend could compare it against the computed $350.00 before executing and reject with a 409.
- **Frontend (`use-settlement.ts`):** A pre-flight re-fetch before calling `runSettlement` could detect the new total of $350 versus the cached $50 and present the admin with a warning and updated figures.
- **Frontend (`SettlementReview.tsx`):** The success screen could display both the approved total and the executed total side by side, making the divergence immediately visible after the fact.

### Q2.4 — Financial Harm, Not Just a UX Problem

This is a financial correctness problem. Consider: ADJ-004 was added to reverse a penalty. If instead the second adjustment was an additional penalty (−$200 rather than +$300), the actual settlement would be less than what the admin approved. Bob would receive less money with no audit trail in the UI capturing that the admin was not presented the correct figure.

A more harmful inversion: an admin is reviewing a freelancer whose computed total is $50 — they knowingly approve a small payout. Between load and confirm, a large legitimate adjustment is added (say +$5,000). The backend settles for $5,050. The admin's approval at $50 was not informed consent for a $5,050 payment. The business has now disbursed $5,050 based on an admin action that authorized $50. This is not a UX inconvenience — it is a control failure in a financial workflow.

---

## Q3: Predict the Failure Mode

### Q3.1 — Database State After `createRemittance` Throws

WL-003: `status = 'SETTLED'`, `last_settled_at` = set.
WL-004: `status = 'SETTLED'`, `last_settled_at` = set.
No remittance record exists.
No remittance line items exist.
Bob has been paid **$0** — the payout depends on the remittance record which was never written.

### Q3.2 — Frontend State After 500 Response

`confirmSettlement` catches the error and executes:

```typescript
setError("Settlement failed. Please try again.");
setIsSettling(false);
```

`setWorklogs([])` is inside the `try` block and was never called. The worklogs array in state is unchanged — WL-003 and WL-004 remain displayed with their amounts. `previewTotal` is still $50.00. The admin sees the error banner at the top and the same table they were looking at. The natural action is to click Confirm again.

### Q3.3 — The Retry

`runSettlementForUser("USR-002")` executes again. `db.getOpenWorkLogsByUser("USR-002")` queries `WHERE status = 'OPEN'`. Both WL-003 and WL-004 are now SETTLED. The query returns an empty array.

`worklogs.length === 0` → the function returns `null`.

In `api.ts`, the null branch executes:
```typescript
res.json({ message: "No open worklogs found for this user." });
```

HTTP 200 is returned with no `remittance` field.

### Q3.4 — Frontend Crashes on the Retry Response

In `use-settlement.ts`, `confirmSettlement` does:

```typescript
const result = await runSettlement(userId);
setSettlementResult({
  remittanceId: result.remittance.id,
  totalAmount: result.remittance.totalAmount,
});
```

`result` is `{ message: "No open worklogs found for this user." }`. `result.remittance` is `undefined`. Accessing `.id` on `undefined` throws a `TypeError` at runtime. The `catch` block catches it and sets `error = "Settlement failed. Please try again."`. The UI shows the same error banner again, with the same stale worklogs still visible in the table.

### Q3.5 — Full Compound State

**Database:** WL-003 and WL-004 are SETTLED. No remittance record exists for either run. Bob has received no payment.

**In-memory (`settledWorkLogIds`):** Contains `"WL-003"` and `"WL-004"` — populated during the first (failed) run. On the second run, these IDs would have been skipped even if the worklogs were somehow still OPEN, which they are not.

**Frontend:** Error banner reads "Settlement failed. Please try again." The table still shows both worklogs. The admin is in a loop — every retry produces the same result.

**New admin, fresh session:** A new `SettlementReview` for Bob calls `fetchWorklogs("USR-002", "OPEN")`. The query returns zero rows. The component renders "No open worklogs to settle." The new admin sees nothing to act on and has no indication that anything went wrong.

**Recovery path:** There is no normal operation path to recover. The worklogs are permanently SETTLED with no remittance. Manual database intervention is required: either reset WL-003 and WL-004 to OPEN status and restart the in-memory Set (requiring a process restart or code change), or manually create the remittance record. Neither is possible through the application's exposed API. This is the direct consequence of performing writes outside a transaction.

---

## Q4: What Happens With THIS Input?

### Q4.1 — Step 1: WL-002 Status After Segment POST

`POST /api/worklogs/WL-002/segments` inserts the new segment (TS-new: 1.5 hrs × $75). Then `db.reopenWorkLog("WL-002")` executes `UPDATE work_logs SET status = 'OPEN' WHERE id = 'WL-002'`. WL-002's status is now **OPEN**.

Note: the POST handler creates its own `Pool` instance inline rather than using the shared `db` module. This is a resource management bug — a new pool is instantiated per request — but does not affect correctness for this trace.

### Q4.2 — Step 2: GET Returns Both WL-001 and WL-002

`db.getOpenWorkLogsByUser("USR-001")` queries `WHERE user_id = 'USR-001' AND status = 'OPEN'`. Both WL-001 (always OPEN) and WL-002 (now OPEN after reopen) are returned.

### Q4.3 — Step 3: Amount Calculation

**WL-001:**
- TS-001: 4 × $75 = $300
- TS-002: 3.5 × $75 = $262.50
- No adjustments
- Total: **$562.50**

**WL-002:**
- TS-003: 6 × $75 = $450
- TS-004: 2 × $75 = $150
- TS-new: 1.5 × $75 = $112.50
- ADJ-001: −$150
- Total: 450 + 150 + 112.50 − 150 = **$562.50**

### Q4.4 — Step 4: Review Screen

| WorkLog | Task | Amount |
|---------|------|--------|
| WL-001 | API Integration | $562.50 |
| WL-002 | Bug Fix Sprint | $562.50 |

`previewTotal` = 562.50 + 562.50 = **$1,125.00**. Confirm button: **"Confirm Settlement — $1,125.00"**.

### Q4.5 — Step 5: Settlement Execution

`runSettlementForUser("USR-001")` independently calls `getOpenWorkLogsByUser` and `calculateWorkLogAmount` using the same `getTimeSegments` and `getAdjustments` functions. It computes identical amounts: WL-001 = $562.50, WL-002 = $562.50. Remittance total = **$1,125.00**.

### Q4.6 — Step 6: The Ledger

- REM-001 (January): $450.00 paid (COMPLETED)
- REM-002 (February): $1,125.00 paid

**Total paid to Alice: $1,575.00**

**Correct total (all segments, all adjustments, no double-counting):**

WL-001: (4 + 3.5) × $75 = $562.50
WL-002: (6 + 2 + 1.5) × $75 − $150 = $712.50 − $150 = $562.50
Grand total: $562.50 + $562.50 = **$1,125.00**

**Overpayment: $1,575.00 − $1,125.00 = $450.00**

The $450 overpayment equals REM-001 exactly — TS-003 and TS-004 were settled in January but included again in the February calculation because `calculateWorkLogAmount` receives all segments from `getTimeSegments` with no filter relative to `lastSettledAt`.

### Q4.7 — Step 7: Silent Agreement Is Worse

When preview and settlement agree, the admin receives false confidence. A $1,125 preview matching a $1,125 settlement looks correct — the admin has no reason to scrutinize further. But the number is wrong in exactly the same way for both the preview and the backend: both include the previously-settled segments. There is no anomaly signal anywhere in the stack. The discrepancy in Q2 was at least detectable — an alert on divergence between preview and execution would have caught it. Here, both sides agree on the wrong number. No comparison-based guard will fire. The overpayment will only be discovered through external reconciliation against payment records, if it is discovered at all.

---

## Q5: Fix Evaluation

### Q5.1 — Fix A: Which Segments Does `filterNewSegments` Keep?

`worklog.lastSettledAt` = `"2025-01-31T18:00:00Z"`.

`filterNewSegments` applies `new Date(segment.createdAt) > new Date(lastSettledAt)`:

- TS-003 (Jan 12): `Jan 12 > Jan 31` → **false, excluded**
- TS-004 (Jan 13): `Jan 13 > Jan 31` → **false, excluded**
- TS-new (February): `Feb > Jan 31` → **true, kept**

Only the new segment is passed to `calculateWorkLogAmount`.

### Q5.2 — Adjustments Are Not Filtered — Alice Is Still Underpaid

`calculateWorkLogAmount(newSegments, adjustments)`:

- newSegments: TS-new only → 1.5 × $75 = $112.50
- adjustments: ADJ-001 (−$150) — not filtered, applied in full

Result: $112.50 − $150.00 = **−$37.50**

Alice is paid **−$37.50** for WL-002 in February. The correct amount for new work only is $112.50 (no prior adjustment should apply to work submitted after the previous settlement). ADJ-001 was already accounted for in the January settlement where it correctly reduced the $600 gross to $450. Applying it again in February double-counts the deduction.

Fix A solves the segment double-count but introduces adjustment double-count, creating a new underpayment bug. A complete fix requires filtering adjustments by `createdAt > lastSettledAt` symmetrically with segment filtering, or — better — recording which adjustments were included in each remittance line item.

### Q5.3 — Strict Greater-Than Boundary Edge Case

If a segment's `createdAt` timestamp equals `lastSettledAt` exactly (same millisecond), `filterNewSegments` excludes it. The segment was created at the settlement cutoff moment, meaning it may not have been included in the prior settlement (depending on race conditions in that run's query timing), yet it is also excluded from the current one. The segment is silently orphaned — never settled.

In practice, this requires a segment to be inserted within the same millisecond as the settlement's `last_settled_at` write. This is unlikely in normal operation but possible under high load or in test environments with mocked clocks. The financial impact is a missed payment for that segment — the freelancer is underpaid by that segment's amount with no error raised anywhere in the system. The fix is straightforward: use `>=` if the intent is to exclude only segments confirmed as part of the prior settlement via an explicit record, rather than relying on timestamp coincidence.

### Q5.4 — Fix B: Does It Prevent the Q2 Stale-Data Problem?

Yes, for the specific scenario in Q2. The pre-flight re-fetch would compute `freshTotal` using the database state that now includes ADJ-004. `freshTotal` would be $350. `previewTotal` is $50. `|350 − 50| = 300 > 0.01` — the condition triggers. The warning is shown, the worklogs state is updated to the fresh data, and the admin is prevented from confirming the stale total. The guard works as intended for this case.

### Q5.5 — TOCTOU Race Condition Between Pre-Flight and Execution

Yes. Between `fetchWorklogs` (the pre-flight) and `runSettlementForUser` executing on the server, data can change again. This is a Time-of-Check to Time-of-Use (TOCTOU) race condition. The window is small — milliseconds in typical operation — but in a financial system, "small window" is not the same as "safe." A system processing thousands of concurrent settlements, or one where adjustments are applied programmatically and frequently, will eventually hit this window. Financial systems must be correct under concurrency, not just under low load. The pre-flight check reduces the probability of a discrepancy but cannot eliminate it.

### Q5.6 — Do the GET Handler and `settlement.ts` Compute Amounts the Same Way?

No, and this is a significant structural problem. The GET handler in `api.ts` computes amounts inline:

```typescript
const amount =
  segments.reduce((sum, s) => sum + s.hours * s.ratePerHour, 0) +
  adjustments.reduce((sum, a) => sum + a.amount, 0);
```

`settlement.ts` uses `calculateWorkLogAmount` from `calculations.ts`, which delegates to `calculateSegmentAmount` per segment. The result is algebraically equivalent for current inputs, but the logic is duplicated — not shared. If `calculateWorkLogAmount` is modified (e.g., to apply tax, apply a cap, or change segment pricing), the GET handler will silently diverge. Fix B's pre-flight comparison relies on the GET handler and the settlement engine computing the same amounts, but this guarantee does not exist structurally. The GET handler must call `calculateWorkLogAmount` rather than re-implementing the formula.

### Q5.7 — The Fundamental Design Flaw

Neither fix addresses the root cause: **the system has no record of which segments and adjustments were included in a given settlement.**

The `remittance_line_items` table records a `work_log_id` and an `amount` per line item, but not which segments or adjustments contributed to that amount. When a worklog is reopened for a new settlement cycle, the system has no way to know which of its segments have already been paid. `lastSettledAt` is a single timestamp approximation of this boundary — it works only when segments are strictly created after settlement, and breaks as shown in Q4.

The structural fix required:

**Data model:** Add a `settlement_segment_items` table linking each remittance line item to the specific `time_segment` IDs that were included, and a `settlement_adjustment_items` table for adjustments. Each segment and adjustment should carry a `settled_in_remittance_id` nullable foreign key, set atomically when settled.

**Backend:** `settlement.ts` must filter segments to those where `settled_in_remittance_id IS NULL`, not those where `createdAt > lastSettledAt`. Adjustments are filtered identically. After settlement, the engine updates `settled_in_remittance_id` on each included segment and adjustment within the same transaction as `createRemittance`.

**Frontend:** The preview amounts computed in the GET handler must use the same filter — segments where `settled_in_remittance_id IS NULL` — ensuring the preview reflects exactly what the settlement will include.

This change makes double-counting structurally impossible: a segment with a non-null `settled_in_remittance_id` cannot be included in any future settlement regardless of timing, server restarts, or concurrency. The `lastSettledAt` timestamp and `filterNewSegments` utility can be removed entirely as they become redundant.