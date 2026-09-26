# Miida: Phase 1c (fix verification bugs + get the regression suite honest)

> Paste everything below the line into the Base44 builder. Same NON-NEGOTIABLE RULES
> and REPORT FORMAT as the main brief apply. Restore point before this phase:
> checkpoint `6ab84c9bc9b90541844bb2b2`.

---

## CURRENT PHASE: 1c

An independent review ran the regression suites against the code from just before
Phase 1 (commit `a58ba01`) and against the current code (`e5215bc`). Fix everything
below. Change nothing else. Do not change any User record.

### 1c.1 `retell-confirm-code` reports success even when it didn't record it (fail-open)
**File:** `base44/functions/retell-confirm-code/entry.ts`
**Problem:**
- On a correct code, the conditional `updateMany(... { $set: { caller_verified: true ... } })`
  result is ignored, and errors are swallowed.
- If the update matches 0 records (a parallel request won, or the conditions didn't
  match) or throws, the function still returns `verified: true`.
- The agent then proceeds, but `retell-log-support-message` rejects because
  `caller_verified` is not true. Two parallel correct guesses both return
  `verified: true`.

**Fix:**
- Check the `updateMany` result. Return `verified: true` only when exactly one record
  was updated.
- On 0 updated, or on an error, return
  `{ verified: false, reason: "verification_conflict_retry" }`.
- Never swallow the error into a success.

### 1c.2 Parallel wrong guesses are not counted
**Problem:** a wrong guess increments only if `verification_attempts` still equals the
value read earlier. When N guesses arrive together, one increments and the rest are
silent no-ops. So far more than 3 guesses are possible.

**Fix:** reserve an attempt BEFORE comparing the code:
1. Call `updateMany({ id, caller_verified: { $ne: true }, verification_attempts: { $lt: 3 } }, { $inc: { verification_attempts: 1 } })`.
2. If it updated 0 records, return `too_many_attempts` without comparing.
3. Only then hash and compare. A correct code sets verified with the checked update
   from 1c.1.

### 1c.3 Re-requesting a code inside the same call resets the limits
**Problem:**
- The issuance cap counts `MiidaCallRecord` rows per `caller_email`, but there is one
  row per call.
- Calling `retell-verify-client` again in the same call overwrites that row with a new
  code and resets `verification_attempts` to 0.
- So one call can request unlimited codes, with 3 fresh guesses each.
- The cap check also fails open: if the lookup throws, issuance continues.

**Fix:**
- Record every issued code as its own row. Reuse `VerificationChallenge` or add a
  `CallerVerificationIssue` entity with service-only rules: `caller_email`,
  `retell_call_id`, `issued_at`.
- Enforce 3 per hour and 10 per day per account from those rows.
- Also enforce at most 2 issuances per `retell_call_id`.
- Issuing a new code must never reset the attempt count for that call. Attempts are
  counted per call, not per code.
- If the cap lookup fails, refuse to issue (fail closed).

### 1c.4 Two test suites broken by Phase 1, and one false "pre-existing" claim
`scripts/test-notify-conversation-auth.mjs` and `scripts/test-email-headers.mjs`
**passed before Phase 1 and fail now.** The Phase 1b report called them pre-existing.
That is incorrect.

- `test-notify-conversation-auth`: `internalAuth.ts` now imports `./timingSafe.ts`, but
  the harness sandbox provides no `require` for it (`ReferenceError: require is not
  defined`). The internal-secret tests no longer run at all.
- `test-email-headers`: `send-automated-email` now imports `workspaceGuard.ts`, which the
  harness doesn't stub (`Unexpected test import: ../../shared/workspaceGuard.ts`). The
  email header-injection (CRLF) tests no longer run.

**Fix:** update both harnesses to load or stub the new imports so every original
assertion runs and passes again. Do NOT delete, skip, or weaken any assertion. Add one
assertion to `test-notify-conversation-auth` proving that a wrong secret of the right
length is rejected.

### 1c.5 Nine suites were already failing before Phase 1: triage every one
These failed at commit `a58ba01` (before any security work). Per
`src/docs/launch-status.md`, all 70 suites passed on Sep 17, so something between
Sep 17 and Phase 1 broke them. For EACH suite, decide whether it's:
- **(A) the test is stale:** the harness is missing a new import, or it uses a
  hard-coded date. Fix the harness/fixture. Keep every assertion's intent.
- **(B) the app regressed:** the code no longer does what the test protects. Fix the
  code, not the test.

| Suite | Current first failure |
|---|---|
| `scripts/test-billing-pause.mjs` | `Unexpected import ../../shared/membershipCatalog.ts` |
| `scripts/test-miida-growth-funnel.mjs` | `Unexpected test import: npm:@base44/sdk@0.8.48 in book-audit` |
| `src/scripts/test-audit-connected.mjs` | `This time no longer meets the minimum booking notice` (likely a date-dependent fixture: make it relative to "now") |
| `src/scripts/test-checkout-journey.mjs` | `new paid buyer reaches registered authenticated setup...` assertion mismatch |
| `scripts/test-invoice-return-journey.mjs` | `invoice provider-confirmed terminal unpaid is a terminal outcome, not indefinite waiting` |
| `scripts/test-prompt22-verification.mjs` | `E1 receptionCancellation mirrors verify same workspace and owner before mutating` |
| `src/components/billing/tests/cancellationRegression.mjs` | `soft cancellation followed by repeated deletion stays pending` (expected 202) |
| `scripts/test-support-attachments.mjs` | `AttachmentPreview ... never reuses a cached URL` (`ensureFresh function present`) |
| `src/scripts/test-support-authority.mjs` | `filterClientConversationPatch validates priority is a bounded integer 0-5` (`Field not updatable by client: priority`) |

Several of these protect billing, cancellation, and client-data behavior, so treat
(B) as likely until proven otherwise. For each one, report A or B, the root cause in
one sentence, and what you changed.

### 1c.6 Tests for 1c.1–1c.3
Add to `scripts/test-phase1-isolation.mjs`:
- A zero-updated verify returns `verified: false`.
- 5 parallel wrong guesses leave the call locked after 3.
- A third code request in the same call is refused.
- Re-issuing a code doesn't reset attempts.
- A failed cap lookup refuses issuance.

### Done when
`node scripts/run-regression-tests.cjs --all` reports **0 failed suites**. Include its
final summary line in the report. Also report every `release:check` stage with its exit
code. Then create a checkpoint and report its real ID.
