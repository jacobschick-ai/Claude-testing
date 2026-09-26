# Miida — Pre-Launch Security, Isolation & Portability Brief (for the Base44 builder)

> How to use: paste everything below the line into the Base44 builder. Change the
> "CURRENT PHASE" line each time. Run one phase per turn, review its report, then
> send the next phase.

---

## CURRENT PHASE: 1

You are working on Miida, a managed business-workspace platform for small service
businesses (Essentials $60/mo, Pro coming). Every client gets one Miida-hosted
workspace (SubAccount). Before we launch paid memberships we must be able to promise
clients three things truthfully:

1. **Isolation:** a workspace's data is only ever visible to that workspace's owner, the
   owner's invited members, active Elder Heads, and staff the client has explicitly
   consented to, and nobody else.
2. **Portability:** a client can export their complete workspace at any time in a
   stable, versioned format that a new website or database can import.
3. **Live connection before severance:** a client's own website can sync with their
   workspace through a secure, versioned API until they fully sever.

Work **only on the CURRENT PHASE**. Then stop and report (see "Report format").

---

## NON-NEGOTIABLE RULES (apply to every phase)

- **Authority model** (already defined in `src/docs/managed-launch-requirements.md` A1):
  - Elder Head = `team_title === "head"` AND `full_access === true` AND active. Use the
    existing `isElderIdentity()` / `accountActive()` in `base44/shared/staffIdentity.ts`.
  - `role === "admin"` alone, `team_title === "head"` alone, `is_owner`, or a staff
    assignment alone must **never** grant access to client workspace data or client
    credentials.
  - Non-Elder staff (Head, Manager, Agent, Marketer) need **all** of: the page, the
    action capability, assignment to that workspace, and **current client consent**.
    Reuse the existing consent/grant system (`base44/shared/clientConsent.ts`,
    `authorizeSupportAccess`, `AccessRequest`, `src/lib/resourceAccess.js`). Do not
    invent a second consent system.
- **Authorize before you read.** Look up the caller and check authority *before* fetching
  the target record, so that "not found" and "forbidden" look the same to unauthorized
  callers.
- **Re-read the caller from the database** (`asServiceRole.entities.User.get(me.id)`)
  before any privileged decision. Never trust session claims.
- **Do not:** enable billing, publish the app, change prices, send real messages, charge,
  cancel, or delete any customer record or file. Do not weaken or skip any existing test.
- **Migrations that touch live records** must ship as a dry-run first (counts + sample
  IDs, no writes). Only run the real migration after the owner approves the dry-run.
- Keep changes minimal and match existing code style. Reuse shared helpers.
- After every phase run `npm run release:check`. Every stage must pass. A skipped or
  red stage is not a pass.
- Create a checkpoint when the phase is complete.

---

## PHASE 1: Close known data leaks (must fix before launch)

### 1.1 Workspace export leaks everything to the wrong people
**File:** `base44/functions/export-workspace-data/entry.ts`
**Problems:**
- (a) The "owner" check is `me.sub_account_id === targetId`. Invited workspace
  *members* have that same `sub_account_id`, so any member can export the whole workspace.
- (b) `isStaffAdmin()` allows `role === "admin"`, so any ordinary Head can export **any**
  client's workspace with no assignment or consent. It also trusts `me.is_owner`, which is
  not a protected User field.
- (c) The export includes `ClientWebhook` rows with raw `token`, `environment_key`, and
  `service_key` values. Whoever holds the zip can impersonate the client's website.

**Why:** this is the single fastest way for client data to leave Miida without the
client knowing.

**Fix:**
- Allowed exporters: the workspace **owner** (resolve with the same owner-resolution
  helper used in `manage-workspace-member`), users listed in
  `SubAccount.export_delegated_user_ids`, or an active Elder Head. Nobody else.
- Remove the `role === "admin"` and `is_owner` paths.
- Exclude every secret field from every exported entity: tokens, token hashes, keys,
  confirmation tokens, and verification fields. Build this as an explicit denylist
  constant and document it.
- Return 404 for both "doesn't exist" and "not allowed".
- Add regression tests: member denied, ordinary Head denied, Elder allowed, delegate
  allowed, owner allowed, no secret fields in the output.

### 1.2 Privilege checks that treat any Head/admin as an Elder
**Problem:** about 25 backend functions authorize with `role === "admin"`,
`role === "admin" || team_title === "head"`, or treat
`role === "admin" && !team_title` as Elder. Ordinary Heads use `role = "admin"`, so
they get Elder-level power. Known instances (sweep for more):
`manage-marketing-credentials` (client ad/marketing credentials), `send-direct-email`,
`send-automated-email`, `provision-email-domain`, `provision-sending-domain`,
`check-sending-domain`, `verify-email-domain`, `decouple-sending-domain`,
`provision-mida-primary-domain`, `verify-mida-primary-domain`,
`decouple-mida-primary-domain`, `update-workspace-domain`, `ensure-mida-hub-workspace`,
`update-client-addons`, `enable-client-feature`, `sync-workspace-features`,
`reconcile-workspace-access`, `get-workspace-profile`, `get-mida-pulse`,
`add-admin-host`, `verify-assignment`, `manage-escalation-flow`, `send-boom`,
`collect-ses-metrics`, `archive-monthly-call-costs`, `ask-admin-ai`,
`manage-meta-marketing`, `manageAIReceptionConfig`, `archiveClientReceptionCosts`,
`start-client-conversation`, `log-workspace-visit`, `record-shadow-activity`,
`list-team-members`, `set-member-display-name`, `publishMembershipPresentation`.

**Why:** any ordinary Head could send email as a client, read client marketing
credentials, change client domains, or toggle client features. That breaks our own A1
rule.

**Fix:**
- Create **one** shared authority module, e.g. `base44/shared/workspaceGuard.ts`,
  exporting:
  - `requireActiveUser(req)`: re-reads the caller from the DB.
  - `isElder(user)`: wraps `isElderIdentity` + `accountActive`.
  - `authorizeWorkspaceAction(base44, user, subAccountId, actionKey)`: returns
    allow/deny with a reason. Owner/member rules, Elder bypass, and for non-Elder staff:
    page + action + assignment + current consent.
  - `authorizeInternalStaffAction(user, actionKey)`: for Miida-internal pages that
    touch no client data (e.g. Miida's own sales pipeline).
- Replace every ad-hoc check above with the guard.
- Remove the `role === "admin" && !team_title => elder` shortcut everywhere. If any
  real Elder account relies on it, list those accounts in the report instead of
  guessing, so the owner can set `team_title = "head"` and `full_access = true`
  properly.
- If a function is platform-only and touches no client data, say so in a one-line
  comment and use `authorizeInternalStaffAction`.

### 1.3 Client data tables readable by all staff without consent
**Problem:** these entities' read rules let any ordinary Head and/or Manager read
**every** client's rows:
- `MiidaCallRecord`: AI call records with caller name, email, phone (Head, Manager).
- `AIReceptionLine`: every client's AI phone line config and client email (Head, Manager).
- `SeveredArchive`: every severed client's archive (Head).
- `ObservationAudit` (Head).
- `Service`, `Booking`, `Client`, `Deal`, `Activity` (Head, Manager; see Phase 2).
- `ClientFeatureOverride` (Head).

**Why:** the owner's rule is that Managers must **not** see any workspace's AI call
records, including assigned workspaces, unless the client has granted consent. The same
applies to all client data.

**Fix:**
- Direct entity read and write for these tables: Elder Head, the record's own
  workspace owner/members where applicable, and service role only.
- Staff access goes through a backend function that calls
  `authorizeWorkspaceAction(..., consent required)`.
- Update UI pages that list these tables to call the mediated function. Pages must
  show a clear "Request client access" state, not an empty list or an error.

### 1.4 AI phone "verification" accepts just an email address
**Files:** `retell-verify-client`, `retell-log-support-message`, `base44/shared/retellAuth.ts`
**Problem:** anyone who calls and states a client's email is treated as that client. The
workspace ID and open conversation are returned, and their words are logged into the
client's real support thread as `sender_type: "client"`.

**Why:** caller impersonation. It pollutes a client's support history and could trick
staff into acting on fake requests.

**Fix:**
- Two-step verification. `retell-verify-client` sends a 6-digit one-time code by SMS
  to the phone on file (fallback: email on file), using the existing
  `VerificationChallenge` / `send-verification-code` patterns. Codes are hashed, expire
  in 10 minutes, allow at most 5 attempts, and are bound to `retell_call_id`.
- A new `retell-confirm-code` endpoint returns `verified: true` plus a short-lived
  call-scoped session ID.
- `retell-log-support-message` must accept that session ID and **ignore** any
  caller-supplied `sub_account_id`. The workspace comes from the verified session only.
- Until verified, log messages as `sender_type: "unverified_caller"`, visible to staff
  with a warning badge.
- In the report, list the exact changes the owner must make to the Retell agent's call
  flow (the agent must ask for the code).

### 1.5 Internal workflow secret compared in a timing-unsafe way
**File:** `base44/shared/internalAuth.ts`
**Problem:** `body.__secret === secret` can leak the secret through response timing.
**Fix:** use a constant-time comparison. Reuse the one in `retellAuth.ts`, moved to a
shared helper. Reject values over 4096 characters.

### 1.6 Unprotected `is_owner` flag
**Problem:** `is_owner` is checked in several functions (`ask-admin-ai`,
`manage-marketing-credentials`, `admin-client-action`, `export-workspace-data`) but it
is not a declared, write-protected User field.
**Fix:**
- Stop using `is_owner` for authorization anywhere.
- Confirm in the report whether `base44.auth.updateMe({ is_owner: true })` can set it
  today. If it can, block it by declaring the field with an Elder-only write rule, and
  list any accounts that currently have it set.

### Phase 1 done when:
- No function grants client-data or credential access based on `role === "admin"`,
  `team_title === "head"` alone, or `is_owner`.
- All tables in 1.3 are Elder/owner/service-only for direct access.
- Tests cover every change above, and `npm run release:check` is fully green.

---

## PHASE 2: Every piece of client data belongs to exactly one workspace

**Problem:** `Client`, `Deal`, `Activity`, `Booking`, `Service`, `AIReceptionLine`,
`EscalationFlow`, and `SendingDomain` have no `sub_account_id`. They mix **Miida's own
sales data** (e.g. `Prospects.jsx` and `AuditBridgeWidget.jsx` create Deals) with
**clients' workspace CRM data** (`CRMDashboard`, `PipelineBoard`, `ActivityFeed` filter
by `created_by_id`). As a result:
- staff can read every client's CRM,
- a client's own team members can't see each other's deals,
- workspace export silently skips these tables.

**Why:** isolation and export cannot be promised while client data isn't stamped with
its workspace.

**Fix:**
1. Add to each table: `sub_account_id` (string) and `workspace` (enum:
   `"miida_internal"` | `"sub_account"`), following the pattern `Contact` already uses.
2. Write rules: `workspace = "sub_account"` rows follow the same read/write rule as
   `Contact`. `miida_internal` rows are visible only to Miida staff with the matching
   internal page/action.
3. Update every create call (frontend and backend) to stamp `sub_account_id` and
   `workspace` server-side. Never trust a client-supplied workspace. Reject mismatches.
4. Update every list/filter call to filter by workspace instead of `created_by_id`.
5. **Migration, dry-run first:** a backfill function that assigns each existing row to
   a workspace:
   - via `client_id` → the Client's workspace;
   - via `created_by_id` → that user's `sub_account_id`;
   - rows created by Miida staff with no client link → `miida_internal`;
   - anything ambiguous → `unassigned` and **not visible to anyone except Elders**.
   Output counts per table and per category, plus up to 20 sample IDs each. **Do not
   write until the owner approves.**
6. Add a build-time check (a script in `scripts/`, included in `release:check`) that
   fails if any entity holding client data lacks `sub_account_id` plus a rule that
   references it. Maintain an explicit allowlist of global/platform tables with a
   one-line reason each.

### Phase 2 done when:
The dry-run report is delivered, and after approval the migration completes with zero
`unassigned` rows (or a named list for the owner). The build check passes.

---

## PHASE 3: Prove isolation (the evidence behind "high assurance")

**Problem:** `src/docs/launch-status.md` records that native enforcement is not proven.
Only mocked tests exist.

**Why:** we can't honestly promise isolation until real accounts fail to cross
workspaces.

**Fix:**
1. `scripts/isolation-matrix.mjs`: a test harness that signs in as each test account
   and, for **every** entity holding client data and **every** backend function taking
   a workspace/user/record ID, attempts read, list, create, update, delete, and export
   against the *other* workspace. Every attempt must be denied. Every legitimate action
   must succeed.
2. Test identities (the **owner will create these real accounts** and store their
   credentials as app secrets; list exactly what's needed):
   - Elder Head,
   - ordinary Head,
   - Manager,
   - Agent,
   - Workspace A owner,
   - Workspace A member,
   - Workspace B owner.
   Include a consent-granted case and a consent-revoked case.
3. Output a matrix report (`src/docs/isolation-matrix-report.md`): entity/function ×
   identity × action → expected vs actual.
4. Add a `release:isolation` stage that runs the matrix when the test secrets are
   present. If the secrets are absent, it reports **red** ("not proven"), never green.

### Phase 3 done when:
The matrix runs against the two real test workspaces with zero unexpected allows and
zero unexpected denies.

---

## PHASE 4: Portable Workspace Package (export built for import)

**Problem:** the current export skips tables, caps at 10,000 records per table, builds
the zip in memory and returns it as one base64 response (this fails for large
workspaces), omits uploaded files, and has no format version.

**Why:** clients who graduate need a complete, importable copy, and so does any future
per-client database.

**Fix:**
1. Define `miida-package` **v1**, documented in `src/docs/workspace-package-v1.md`:
   - `manifest.json`: package version, schema version, workspace ID/name, exported_at,
     exporter, per-entity record counts, and a SHA-256 checksum per file.
   - One `entities/<Entity>.jsonl` file per client-data entity (every Phase 2
     workspace-stamped table), with stable IDs and relationships preserved.
   - `files/`: uploaded files belonging to the workspace, plus `files/index.json`.
   - No secrets (reuse the Phase 1 denylist).
2. Build it as a **background job**: create a `WorkspaceExportJob` record, page through
   everything with no silent cap, write the package to private file storage, and
   return a short-lived signed download link. Show progress in Settings.
3. Record every export in `WorkspaceExportAudit` (already exists) and notify the
   workspace owner by email whenever anyone other than the owner exports.
4. A reference importer, `scripts/import-package.mjs`, that validates the manifest and
   checksums and loads a package into an empty target. Test: export test workspace A,
   import it into a clean target, and every count and relationship matches.

### Phase 4 done when:
The round-trip test passes on a workspace larger than 10,000 records in at least one
table, with files included.

---

## PHASE 5: Versioned API for clients' websites before severance

**Problem:** the Hub bridge (`get-hub-passport`, `get-hub-bridge-full-sync`,
`get-hub-bridge-updates`, `receive-hub-bridge`, `client-webhook-ingest`) works but:
- one token can read and write everything,
- tokens are stored in plaintext next to their hash,
- sync endpoints allow any website origin (`Access-Control-Allow-Origin: *`),
- `client-webhook-ingest` accepts `?token=` in the URL,
- there are no rate limits,
- there is no versioning.

**Why:** this API is the long-term promise to clients. Changing it later without
versioning breaks their websites, and one leaked all-powerful token exposes a whole
workspace.

**Fix:**
1. Serve it as `/v1` using the **same entity shapes as `miida-package` v1**, so export
   and live sync never diverge.
2. Scoped API keys per workspace:
   - scope `read` / `write` / `forms_only`,
   - optional per-feature limits,
   - expiry,
   - named label,
   - last-used timestamp.
   Store **only a hash**. Show the full key once at creation. Migrate existing tokens to
   hash-only and keep them working until each client rotates.
3. Client-controlled key management in Settings: create, rotate, revoke. Revocation is
   immediate.
4. An allowed-origins list per key, replacing `*` CORS. `forms_only` keys may keep URL
   tokens (third-party form tools need them), but they can **only** create contacts or
   reviews and never read.
5. Per-key and per-workspace rate limits (reuse `reservePublicRequests`), plus an audit
   log of every API call (key, action, count, and IP hash).
6. Publish `src/docs/api-v1.md`: authentication, endpoints, pagination, errors, rate
   limits, and a deprecation policy.

### Phase 5 done when:
A test website can do a full sync, then incremental sync, then a write, and is cut off
immediately on revocation. A read key cannot write. A forms key cannot read.

---

## REPORT FORMAT (end every phase with this)

1. **Changed:** each file, with a one-line reason.
2. **Tests added:** names, and what each proves.
3. **`npm run release:check` output:** every stage with its exit code.
4. **Owner actions needed:** secrets to add, accounts to create, Retell changes,
   dry-run approvals.
5. **Anything found but not fixed:** with severity and why.
6. **Checkpoint ID.**
