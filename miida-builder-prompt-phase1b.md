# Miida: Phase 1b (finish Phase 1 before Phase 2)

> Paste everything below the line into the Base44 builder. Same NON-NEGOTIABLE RULES
> and REPORT FORMAT as the main brief apply.

---

## CURRENT PHASE: 1b (complete the Phase 1 gaps found in review)

An independent review of Phase 1 found the items below incomplete or incorrect. Fix
all of them. Change nothing else. Do not change any User record.

### 1b.1 MiidaCallRecord must require client consent, not just assignment
**Problem:** the owner's rule is that Heads and Managers must NOT see a workspace's AI
call records, including workspaces assigned to them, unless the client has granted
consent. `base44/entities/MiidaCallRecord.jsonc` currently lets assigned Heads and
Managers read directly. It also exposes `verification_session_token` and
`verification_code_hash`. A 6-digit code hash can be brute-forced instantly, so anyone
who can read the hash can recover the code.

**Fix:**
- Direct read: Elder Head, and the workspace's own owner/members
  (`data.sub_account_id == user.data.sub_account_id` for `role client` or
  `team_title member`). Remove the assigned Head and Manager branches.
- Add field-level read AND write rules to `verification_session_token`,
  `verification_code_hash`, `verification_expires_at`, and `verification_attempts`:
  service role only. Nobody, including Elders, reads them through the entity API.
- Staff access to call records goes through a backend function that calls
  `authorizeWorkspaceAction(base44, user, subAccountId, "/ai-reception:view_calls")`.
  That covers page, action, assignment AND current consent. The function never
  returns the verification fields. Update the UI page that lists call records to use
  it, and show a "Request client access" state when consent is missing.

### 1b.2 Finish the 1.3 tables (Elder-only now, consent-mediated later)
These still let ordinary Heads and/or Managers read every client's rows:
`AIReceptionLine`, `SeveredArchive`, `ObservationAudit`, `Service`,
`ClientFeatureOverride`, `Client`, `Deal`, `Activity`, `Booking`.

(`SeveredArchive` and `ObservationAudit` already have `sub_account_id`.)

**Fix now:**
- Set direct read/update/delete to Elder Head (head + full_access) OR the record's own
  client where a client-binding field exists:
  - `AIReceptionLine.client_user_id == {{user.id}}`
  - `ClientFeatureOverride.user_id == {{user.id}}`
  - Rows the user created in `Client`/`Deal`/`Activity`/`Booking`
    (`created_by_id == {{user.id}}`), so clients' own CRM keeps working until Phase 2
    re-scopes it by workspace.
- Keep create rules unchanged unless they allow the same over-broad access.
- List every UI page that loses data for non-Elder staff as a result. Do not add
  workarounds. Phase 2 moves these to consent-mediated access.

### 1b.3 Functions the sweep missed
- `manage-sandbox-key` and `manage-lead-finder-key`: `!isOwnerUser(me) && me.role !== "admin"`
  lets any admin-role account read/regenerate Miida's internal API keys. Replace with
  `isElder(user)` using the re-read user from `requireActiveUser`.
- `send-support-sms`: any Head/Manager/Agent can text any client. Require
  `authorizeWorkspaceAction(..., "/support:send_sms")` for the target conversation's
  workspace (Elders bypass).
- `get-workspace-profile`: `isMidaStaff` trusts `role === "admin"` or any staff title.
  Staff (non-client) callers must be `isElder` to receive the platform-owner workspace
  profile. Everyone else gets only their own.
- Re-run the sweep: `grep -rnE "role\s*===?\s*['\"]admin['\"]" base44/`. For every
  remaining hit, either fix it or add a one-line comment explaining why it grants no
  access (e.g. labels or diagnostics only). Include the final grep output in the report.

### 1b.4 Harden phone verification
- `retell-verify-client`: generate the code with
  `crypto.getRandomValues(new Uint32Array(1))[0] % 1000000`, zero-padded to 6 digits.
  Never use `Math.random()`.
- Limit code issuance: at most 3 codes per account per hour and 10 per day, counted
  server-side. Return `{ verified: false, reason: "too_many_requests" }` beyond that.
- Bind the verification session to its `retell_call_id`. `retell-confirm-code`,
  `retell-log-support-message`, and `retell-create-audit-booking` must reject a
  session used with a different `retell_call_id`.
- Verified sessions expire 60 minutes after verification (`verified_expires_at`).
  Reject expired sessions.
- `retell-log-support-message`: stop accepting a caller-supplied `sub_account_id`. Use
  ONLY `verifiedRecord.sub_account_id`. If it's empty, reject with 403.
- Make the failed-attempt count safe against parallel guesses. Record each attempt as
  its own row, or re-read and compare before accepting, so parallel requests can't
  exceed 3 attempts.

### 1b.5 Fail closed in the guard
`base44/shared/workspaceGuard.ts` `requireActiveUser`: if the database re-read of the
caller fails, return 503 ("Please try again"). Never fall back to session claims.

### 1b.6 Real checkpoint
Create a real Base44 checkpoint when done, and report its actual ID, not a description.

### Tests
Extend `scripts/test-phase1-isolation.mjs` (and make sure the regression runner
discovers it) to prove each item above:
- A Manager assigned to a workspace, without consent, gets no call records.
- Verification fields are never returned.
- `Math.random` is not used in verification.
- The issuance cap works.
- A session used with a different call ID is rejected.
- Expired sessions are rejected.
- A caller-supplied `sub_account_id` is ignored.
- The guard returns 503 when the DB read fails.

Report the regression runner's pass count, and every `release:check` stage with its
exit code. The frontend and backend typecheck failures pre-date Phase 1; say so
explicitly, but do not claim the release check as green.
