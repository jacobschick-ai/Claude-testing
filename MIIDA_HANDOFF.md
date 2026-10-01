# Miida — Full Context Handoff

Written October 1, 2026, at the end of a long build session. Paste this whole file into the new chat as the first message, or attach it. It covers what Miida is, how it works, what was done, and exactly what to do next.

---

## 0. Read this first (for the AI picking this up)

**The app**
- **Name and ID:** Miida, a Base44 app. App ID `6a5d727d98fe0a5accd77f4b`.
- **Live URLs:** `miida.base44.app` (also `miida.co` and `mid-a.com`). Preview: `preview-sandbox--6a5d727d98fe0a5accd77f4b.base44.app`.
- **How to work on it:** use the Base44 MCP tools:
  - `run_command` (sandbox at `/app`, 60-second limit, so run long jobs detached with `setsid nohup … &` and poll);
  - `read_file`, `edit_file`, `write_file`, `create_checkpoint`;
  - `query_entities`, `update_entity_schema`, `execute_api`.
- **Saving work:** file edits auto-commit, but only a **checkpoint** is restorable. Checkpoint after every unit of work.
- **Latest checkpoint:** "Staff assistant role from stored user; stored record also fills nested user data" (`6abe8142ef5b3b0cb7306b4c`).

**Where this file lives:**
- In the app it's `docs/prelaunch-decisions.md`, the builder's context file. `AGENTS.md` points to it.
- On GitHub it's `MIIDA_HANDOFF.md` (repo `jacobschick-ai/Claude-testing`, branch `claude/cloud-sessions-overview-nwikmy`).
- **Whoever is building must update the app copy after every finished step,** with what changed, new decisions and what's next, until launch.
- When the owner says "handoff", paste the full current file into the reply.

**Owner:** Jacob (jacob.schick.b@gmail.com), the founder. He is not a developer yet and wants to learn the code later. Explain things in plain language.

**Working rules. These are non-negotiable and the owner set every one of them.**
1. **Tell the owner exactly what you will do before doing it, and plan it with him.** After doing it, say what went right or wrong and how to continue.
2. Never ask for passwords. Read secret *names* only, never values.
3. Confirm before changing live data, and before anything outward-facing or hard to undo.
4. Never publish the app or turn on billing without his say-so.
5. No AI model names in commits or code.
6. Don't add things he said not to add: no Turnstile yet, no data-rights digest, no email-confirmation step for forms. Two-step sign-in stays OFF.
7. Email works. Only send real test emails if truly needed, because he gets flooded.
8. Notices to workspaces go out **only** through the Base44 builder or the owner's assistant. Elders can save drafts in the hub but can never send.
9. Before finishing code: run `npm run release:check` in `/app`. It covers lint, frontend and backend type checks, build, 116 regression suites and npm audit. All must pass, then checkpoint.
10. Delete `docs/prelaunch-decisions.md` on launch day. Launch means the app is published AND paid billing is on.

---

## 1. What Miida is and why

- **Mission:** help small and medium businesses become independent and grow, "including from us, if that's ever what you want". Long-term ambition: "the Google or Amazon of AI and AI-management services".
- **Ideal customer:** a business that hasn't integrated AI but could use it to automate parts of the company and make getting paid easier.
- **Core product:** the client **workspace**, an all-in-one business hub with:
  - website management;
  - CRM and pipeline;
  - calendar, jobs and bookings;
  - money and invoices;
  - reputation and reporting;
  - support;
  - AI tools (AI Reception phone line, Conversation AI, Content AI);
  - marketing (an add-on).
- **Business principle:** "We want it to help people, not make them frustrated or feel like they're being ripped off." Every client's data must be secluded, exportable and portable, sustainable short-term and long-term, with high quality and high assurance.
- **Near-term goal:** sign the first few clients to prove the concept is repeatable.
- **Lead Finder AI:** an internal tool only, never sold. The main way Miida gets clients leads is Meta ads.

### Plans and pricing (live `PricingPlan` records)

| Plan | Price | Notes |
|---|---|---|
| Mida Essentials | $59/mo | Main plan. Owner calls it "the $60 plan". |
| Mida Pro | $250/mo | |
| Basic AI Reception | pay-as-you-go | Manual invoice at the actual Telnyx + Retell cost **+50%**. |
| Advanced AI Reception | pay-as-you-go | Same, **+80%**. |
| Maintenance + one-time setup | custom | Custom website/CRM build plus maintenance that scales with complexity. |
| Marketing (add-on) | Meta ad spend + a small % of revenue per job | Money back if no leads in a month. |

- **Launch / founder offer:** "Limited launch offer: the next **5** clients pay **$0 website setup**, just the monthly plan." It lives in `src/components/landing/publicContent.js` as `LAUNCH_CLIENT_COUNT = 5`. No other founder discount exists in the code. Ask the owner if he has more in mind.
- **Free memberships:** right now all clients have a free workspace, as does anyone manually added and given a membership through the Permission Matrix. A free plan can't be cancelled, only removed by deleting the account, unless the free membership is revoked. Staff don't need client plans.
- **One owned workspace per email.** People can join any number of other workspaces as members, with team title `member`, which is treated as a client.
- **Maintenance and management are different things:**
  - **Maintenance:** if Miida didn't build something right, a client with maintenance makes a maintenance call and Miida fixes it. Without maintenance they might spend a feature call.
  - **Management:** a free toggle for every paying client. It gives Miida staff **view-only** access to the workspace at all times. Every *action* still needs the client's approval through the support chat ("Unlock").
  - Turning management on or off sends a confirmation email to the workspace owner. After it's turned off there's a **24-hour lockout**, with a countdown under the toggle.
- **Severing and sunsetting:**
  - **Severing:** a client leaves Miida and takes their data or site to their own independent destination.
  - **Sunsetting:** moving their services, such as the domain and phone line, to providers they own.
  - These are `SeveranceCase` and `SunsettingCase`, with exports, retirement manifests and archives. "Miida Growth" is a further tier/path; a client moving to it keeps their AI Reception number.
- **AI Reception journey:**
  1. The client requests it.
  2. They get a number to sign up officially through the Miida reception line.
  3. Miida agents build the client's Retell agent with a new Telnyx number.
  4. An agent enters it in the activation table under **Manage** on the client's record, which turns on the client's AI Reception page and its cost tracking.
  5. The number stays on that workspace until it's deactivated through Manage (Elders always can; others need the permission). It turns off automatically when the account is deleted or AI Reception is cancelled.
- **Account deletion:** 30-day recovery window, then permanent deletion including Base44's erasure. Some records are kept as long as the law requires.
- **Legal and contact email:** support@miida.co.

---

## 2. People, roles and the hierarchy

- **Staff hierarchy:** Elder Head > Head > Manager > Agent, set by each person's "reports to".
- **Branch caps (enforced):** 10 Managers per Head and 10 Agents per Manager.
- **Elder Head** = `team_title: head` AND `full_access: true`. This is "a completely different level".
- **Current Elders:** 4 accounts: Jacob S, Jacob Schick, and Josh Hernandez (two accounts). **Only these current Heads are Elders.** Any new Head must be an ordinary Head (`full_access: false`), governed entirely by the Permission Matrix.
- **Owner identity:** the owner email in `base44/shared/ownerIdentity.ts` (`OWNER_EMAIL`) is the server-side owner check, by design. It is the only place in the code with that email.
- **Test accounts:**
  - `jacob.schick2k7@gmail.com` is an **Agent** test account (no Calendar yet; see task 3C).
  - The "office" workspace is a test workspace.
- **Head sovereignty:** each level controls what's below it. Heads and Managers can only pass on what their own *delegation budget* allows. Elders can do anything.
- **Oversight:** goes up the chain.
  - Elders see every chat.
  - Heads see their Managers' and those Managers' Agents' chats.
  - Managers see their Agents' chats.
  - An overseer can only reply by taking over, which kicks the agent out. Only the level above has that power, never peers.
- **Elder account lock is builder-only.** The owner says "Lock (Elder) account <email>" (any Elder may lock) or "Unlock (Elder) account <email>" (owner only). The handling is described in `AGENTS.md` and runs the `elder-account-lock` function. There are no lock buttons in the app.
- **Panic Lock** (in Team Access) is for non-Elder staff. It deactivates the account and ends its sign-ins, and unlocking restores their pages exactly.

---

## 3. Permissions and workspaces: how it works now

### Staff pages and buttons
- **Per-role defaults:** the `StartingFeatures` table, edited on the Permissions page's Starting Features grid.
  - A key without a colon is a page (for example `/calendar`).
  - `/route:action` is a button permission (for example `/calendar:edit_appointment`).
- **Per-person settings:** `User.allowed_tabs` (pages) and `User.feature_entitlements` (buttons). Their values mean:
  - `null` / missing: inherit the role's Starting Features;
  - `[]`: everything removed;
  - a list: exactly that list.
- **Server side:** `base44/shared/staffGrantDefaults.ts` `resolvedStaffGrants()`.
- **Browser side:** `src/lib/accessResolver.js` `resolveStaffPage` / `resolveStaffAction`, plus `src/lib/startingFeaturesStore.js`, `src/lib/staffNav.js` and `src/components/WorkspaceShell.jsx`.
- **What's saved live in StartingFeatures today:**
  - Agent: `/support`, `/assignments`.
  - Client Essentials: calendar, jobs, crm, website, reputation, reporting, mida_support.
  - Client Pro: the Essentials set + voice_ai, conversation_ai, content_ai.
  - **Manager, Head and "no plan" clients: nothing saved. No button permissions are saved for any role.** The owner will fill these in himself.

### Client pages
- `src/lib/clientPagePolicy.js` lists the client pages.
- Plan inclusion comes from `PricingPlan.included_features` and StartingFeatures client scopes.
- Per-workspace overrides are in `WorkspaceFeatureAccess`. Per-member overrides (for workspace teammates) are in `WorkspaceMemberAccess`.
- Today some pages are "defaultOn": dashboard, jobs, money, calendar and explore_features. **The owner wants this changed; see task 3.**

### Staff access to a client's workspace (client consent)
- Staff can enter only through an **open support chat they handle and have answered**, and only with the "View Client Workspaces" permission (off by default; per person in the Permission Matrix).
- Management on means view-only. Every action, and every page when management is off, needs the client's **"Unlock"** approval in that chat.
- An approval covers only what the client approved, only for that staff member, and ends when the chat closes or they sign out. Overseers get the same view as the agent.
- **Testing exception:** `ELDER_CLIENT_DATA_BYPASS = true` lets Elders in without approval. **It must be switched OFF before launch.** The Terms already describe the no-bypass behaviour.

### Calendars
- Everyone has their own appointments.
- Heads and Managers see their branch read-only. Elders see and manage everything.
- New appointments always belong to their creator.
- Create, Edit, Cancel and Delete Appointment are separate button permissions, off by default.

---

## 4. Communication

- **Client support chat:**
  - The AI answers until the client calls an agent. After that the chat stays in agent mode, with the AI silent, until it's resolved.
  - Agent alerts go out as a heartbeat to the whole team, with Elders as the fallback. Links point to `/support?conv=<id>`.
- **Staff team chat:**
  - Messages go up only to your direct boss; Heads go to all Elders.
  - Only someone above you opens a chat down to you.
  - Group chats: Manager + Agents, and Head + Managers.
  - Inside Miida a notice pops up and fades. When away, one-to-one messages send an email that never contains the message text.
- **Notices bell** (bottom left in every workspace): update notices, price and terms changes, and a yearly membership reminder. They're sent **only** via the builder or the owner's assistant ("send a notice to all workspaces that says…"). The bell turns gold when there's something new.
- **Email:**
  - Base44 sends basic verification and confirmation emails.
  - Resend (no-reply@miida.co) handles client and transactional email, with bounce and complaint tracking through a signed webhook.
  - Gmail (support@miida.co) is connected.
  - Amazon SES handles client email blasts; its SNS bounce feed isn't set up.
  - Client emails say "automated, do not reply", and clients set their sender display name in Settings.
- **Texting:** Twilio was **removed**. Texting will be rebuilt on **Telnyx** later (+1 830 443 7032, needs 10DLC registration and `TELNYX_API_KEY`). SMS shows as "coming soon".

---

## 5. Security architecture (what exists)

**Who someone is, and what can run without sign-in**
- **Every server function re-reads the caller from the database.** `base44/shared/staffLoginGuard.ts`:
  - `guardStaffLogin` wraps every function's SDK client.
  - `storedCaller()` replaces session values with the stored User record, including the nested `.data` copy.
  - It strips the 12 private fields (`PRIVATE_USER_FIELDS`) and fails closed (503) if the read fails.
- **User field rules:** role, team_title, full_access, pages, buttons, workspace, reports_to, etc. can be written only by an Elder (`head` + `full_access`) or the server. `is_active`, `banned` and `deleted_at` are server-only.
- **Public functions** (24, locked by `tests/test-public-functions.mjs`):
  - signed webhooks: Retell, Telnyx, Wix, Resend, SES, hub bridge;
  - one-time email links;
  - rate-limited public forms: contact, data rights, audit booking, client offer, price list, age confirmation.
- **Bot traps:** a hidden `company_website` field on the contact and data-rights forms.
- **Rate limits:** `base44/shared/publicRequestSecurity.ts`, with the `PublicRequestReservation` table. Every scope used must be in that table's allowed list; `tests/test-reservation-scopes.mjs` enforces this.

**Who can reach which records**
- **hub-records gate:** non-Elders read money, prospects, affiliates, call costs, audits and domains only through the `hub-records` function, which checks the matrix. Key and code tables are server-only.
- **Escalations, assignments and visit logs:** only participants (or Elders) can escalate a record. Flagging requires being able to see the assignment. Clients can only log visits to their own workspace.
- **Staff AI assistant:** the role and workspace come from the stored user. Only real Elders get the web-search model.

**Destructive actions and safeguards**
- **Sealed destructive pass:** account purge and closure require a pass from `base44/shared/authorizedPass.ts`. An import allowlist test (`tests/test-sensitive-imports.mjs`) controls who can import these.
- **Two-step sign-in** for Managers and up is built but OFF (`TWO_STEP_ENABLED`).

**Switches, scans and the latest fixes**
- **Feature switches:** all in `base44/shared/featureFlags.ts`:
  - `ELDER_CLIENT_DATA_BYPASS = true` (turn off before launch);
  - `TWO_STEP_ENABLED = false`;
  - `PAYMENT_FULFILLMENT_ENABLED = false`;
  - `INSTANT_FORM_SYNC_ENABLED = false`;
  - several provisioning/handoff flags, all false.
- **Last Base44 security scan:**
  - No dependency or code findings except the staff-assistant one, now fixed.
  - **Remaining by design or pending:** public forms (Turnstile later), unregistered secret names (Wix, SNS, Meta, Telnyx, Turnstile; registered when set up), the owner email in `ownerIdentity.ts` (by design), and a StartingFeatures rule suggestion (the existing rule is already stricter).
- **Fixed this session:**
  - Values that tables rejected: the contact-form limiter, account deletion (`full_account_deletion`) and teammate invites (`member`). All are now allowed in the live tables.
  - The guest age-confirmation limit (300/hour; **owner wants it smaller**).
  - Privacy and Terms updates: providers, cookies, the staff-access ("Unlock") description, AI providers (Base44 + Google Gemini) and dates.
  - Mobile layout fixes (Money calendar, Staff Overlook, Clients).

---

## 6. Codebase map

- **Location:** `/app` in the Base44 sandbox.
  - Frontend: `src/` (React + Vite).
  - Server functions: `base44/functions/<name>/entry.ts` (Deno, 165 functions).
  - Shared server code: `base44/shared/*.ts` (127 files).
  - Tables: `base44/entities/*.jsonc` (126).
- **Tests:** `tests/` (116 suites).
  - Loaders: `tests/fixtures/loadLocalModule.cjs` (gives fake clients a stored-user fallback) and `tests/fixtures/stubStaffLoginGuard.mjs`.
  - Some tests use their own import maps (for example `test-email-headers.mjs`). A new import in a function may need adding there.
- **Docs:** `docs/setup-guide.md` (outside services), `docs/how-security-works.md`, `docs/prelaunch-decisions.md` (delete at launch). `AGENTS.md` describes the Elder-lock command.
- **Versions:** functions are pinned to `npm:@base44/sdk@0.8.48` (Deno's age policy blocked newer versions); the frontend uses 0.8.52. Deno is a devDependency, used for `typecheck:backend`.
- **Important frontend files:**
  - `src/lib/appRegistry.js`: the page registry, `MANAGEABLE_TABS` and `ALWAYS_TABS`.
  - `src/lib/accessResolver.js`, `src/lib/staffNav.js`, `src/lib/startingFeatures.js`, `src/lib/startingFeaturesStore.js`.
  - `src/components/permissions/*`: the Permission Matrix, the Starting Features grid and the member access panel.
  - `src/components/WorkspaceShell.jsx` (page guard), `src/lib/businessContext.jsx` (profile), `src/lib/hubRecords.js`.
- **Tables by area:**
  - **Identity and access:** User, StartingFeatures, WorkspaceFeatureAccess, WorkspaceMemberAccess, ClientFeatureOverride, AuthorityGrant, AccessRequest, PermissionAuditLog, StaffLoginCheck, VerificationChallenge, VerificationCodeRequest, AgeEligibilityAcceptance.
  - **Workspaces:** SubAccount, ClientPlan, Subscription, WorkspaceTheme, WorkspaceVisitLog, WorkspaceNotice, WorkspaceExportAudit, WorkspaceRetirementManifest, SeveranceCase, SunsettingCase, SeveredArchive, AccountClosureRequest, BusinessRestartRequest.
  - **Client tools:** Contact, Deal, Opportunity, Pipeline, Appointment, Booking, Task, Invoice, RecurringInvoiceCycle, MoneyEntry, MoneyGoal, ReputationReview, Automation, TrackedSite, ClientWebhook, ContactImportFile.
  - **AI Reception:** AIReceptionConfig, AIReceptionLine, ClientAIReceptionLine, ClientAIReceptionCall, ClientAIReceptionCostArchive, MiidaCallRecord, MiidaCallCostArchive, CallerVerificationIssue.
  - **Support and staff:** SupportConversation, SupportMessage, SupportAttachment, InternalConsultation, StaffAssignment, StaffComment, StaffConductReport, TeamChat, TeamChatMessage, TeamNotice, EscalationFlow, EscalationMiss, FiredEmployeeArchive.
  - **Miida hub:** Lead, LeadScan, LeadSettings, LedgerEntry, Referral, Marketer, Payout, AuditBooking, AuditSettings, MidaPrimaryDomain, SendingDomain, EmailIdentity, DeliverabilitySnapshot, PricingPlan, Base44Purchase.
  - **Marketing:** MarketingAccount, MarketingAutomation, MarketingEnquiry, MarketingWorkspacePlan, MetaAdDraft, MetaInstantFormIntake, MetaMarketingConfig.
  - **Market intel (old):** Competitor, CompetitorFinding, MarketIntelSettings, MarketRecommendation, MarketReport.
  - **Email:** QueuedEmail, TransactionalEmailReceipt, SnsWebhookReceipt, Booms (client email campaigns, hidden).

---

## 7. APPROVED NEXT TASKS (owner said "I like the proposed fix" — do these next)

Plan each step with the owner before doing it, per rule 1. Run the release check and checkpoint after each.

### Status log (newest first)
- **Oct 1:** The handoff was put into `docs/prelaunch-decisions.md` (the app's builder context file), and `AGENTS.md` now points to it. Task 1's plan was sent to the owner, proposing **30 per hour** for the guest limit. **Waiting for his go-ahead.** No code for tasks 1–5 has been changed yet.

### Task 1 — Age confirmation (18+)
- **Owner's rule:** only **clients** check the 18+ box, and only when **signing in for the first time** or **booking an audit while signed out**. Nowhere else.
- **Current state:** `requireAgeEligibility` runs in several places:
  - `confirm-age-eligibility`, called from `Login.jsx`, `Register.jsx` and `ClientPortal.jsx`;
  - `src/lib/ageEligibility.js`;
  - `staff-assistant-reply` (context `ask_miida`, which also blocks staff);
  - a `purchase` context;
  - an 18+ dialog that appeared over a client's Support page.
- **Work:** map every caller with `grep -rn "requireAgeEligibility\|confirm-age-eligibility\|AgeVerificationDialog\|eligibilityPayload" src base44`. Remove it from staff paths and from every client path except first sign-in and signed-out audit booking.
- **Guest limit:** the owner wants it **smaller than 300 per hour**. Propose a number (around 30–50/hour), confirm with him, then change `GUEST_CONFIRMS_PER_HOUR` in `base44/functions/confirm-age-eligibility/entry.ts`.
- **Privacy Policy:** add a line that Miida records the 18+ confirmation, with a guest session ID, for signed-out audit bookings and new accounts.

### Task 2 — Cookie consent box (the owner wants it now)
- Show a small box to **first-time visitors**. Every optional category is **OFF by default** until the visitor clicks **Allow**. Add a way to change the choice later, such as a "Cookie settings" link in the footer.
- It must actually work:
  - Gate the client-offer **Meta Pixel** (`src/lib/clientOfferPixel.js`, used by `ClientOffer.jsx`) on consent.
  - Find out whether Base44's built-in analytics (requests to `/api/apps/<id>/analytics/track/batch`) can be paused until consent. Check the Base44 SDK/client config in `src/api/base44Client.js`. If it can't, tell the owner exactly that.
  - Store the choice in localStorage, which counts as necessary.
- Update the Privacy Policy's "Cookies, Local Storage, and Tracking" section to describe the box.

### Task 3 — Permissions: the Permission Matrix is the ONLY source of truth
The owner's rules:
- Every workspace, client or staff, **always** includes **Dashboard and Settings**. Everything else comes only from membership/plan, role (Starting Features) or the Permission Matrix (per person).
- **If nothing is saved for a staff role, members of that role get only Dashboard + Settings** until the owner adds Starting Features. This follows head sovereignty.
- The owner will fill in Starting Features himself.
- **Only the current 4 Heads are Elders.** New Heads are ordinary and governed by the matrix.

Approved fixes:
- **A. One source of truth.**
  - When a role has nothing saved, the Starting Features grid must show exactly what the role gets (Dashboard + Settings only), with a note. Today it falsely shows built-in defaults for Manager and Head.
  - Remove these hard-coded lists:
    - `STAFF_ROLE_DEFAULTS` in `src/lib/accessResolver.js`;
    - `baselineFor()`/`baselinePaths()` in `src/lib/staffNav.js` (used for the "delegated" tracking label in `WorkspaceShell.jsx`; switch that to the resolver);
    - the unused `tabAllowed()` in `src/lib/appRegistry.js`, which allows everything when the list is empty (dangerous if ever wired in), and its re-export in `src/lib/teamTabs.js`;
    - the hard-coded staff menu lists;
    - the `defaultKeysForScope` fallback in `src/lib/startingFeatures.js` for staff scopes.
  - `getStaffStartingPages()` returns an empty Set when nothing is saved, which is correct. Keep the server (`resolvedStaffGrants`) consistent.
  - Check `ALWAYS_TABS` (it includes `/admin`) and `WorkspaceShell` `alwaysStaff` (it includes `/support`) against the "only Dashboard + Settings" rule, and ask the owner about `/admin` and `/support`.
- **Client side:**
  - Today `clientPagePolicy.js` marks dashboard, jobs, money, calendar and explore_features as `defaultOn`. Under the new rule only Dashboard (and Settings) are automatic; the rest come from plan, Starting Features or the matrix.
  - **This changes what existing clients see.** Show the owner exactly which workspaces would lose which pages and agree a migration before changing it.
- **B.** The owner saves Starting Features for each role himself.
- **C. Dead page addresses.**
  - Live data: the Agent test account `jacob.schick2k7` has `allowed_tabs` `["/assignments","/reporting","/appointments","/support","/crm-gen"]`. Replace `/appointments` with `/calendar` and drop `/crm-gen`. This change to live data was approved as part of the proposed fix; tell the owner when it's done.
  - Code: `src/lib/featureConfig.js` points Workflows at `/crm-gen` (should be `/workflows`); `src/lib/staffNav.js` and `src/lib/simpleWorkspace.js` list `/appointments`; `BusinessSnapshotCard.jsx` links to `/appointments` (should be `/calendar`).
  - Make the member-save path (`manage-team-member` / `MemberAccessPanel`) reject unknown page keys and convert `/appointments` to `/calendar`.
- **D. Tests** that, for every role, compare what the grid shows, what the browser grants and what the server (`resolvedStaffGrants`) grants. Include roles with nothing saved, which must give Dashboard + Settings only.
- **Also audit "role: admin" in table rules.** The User schema says ordinary Heads also get `role: 'admin'`, so any table rule that lets `role: admin` in grants new ordinary Heads Elder-like access.
  - For example, `StaffAssignment`'s read rule starts with `{"user_condition":{"role":"admin"}}`.
  - The private verification fields on User are readable by `role: admin`.
  - `canSeeAssignment()` in `verify-assignment` treats `role === 'admin'` as full access.
  - Find every such rule (`grep -l '"role": "admin"' base44/entities/*.jsonc`) and the code checks, and change them to the Elder test (`head` + `full_access`) where Elder-only was meant. Plan it with the owner first.

### Task 4 — Remove code that no longer connects to anything
- The owner wants every leftover removed.
- **Known leftovers:** `ReminderDeliveryTest.jsonc` (supposedly removed earlier but still in `base44/entities`), `tabAllowed`, the hard-coded role lists above, and the `/crm-gen` and `/appointments` references.
- **Old market-intel tables:** Competitor, CompetitorFinding, MarketIntelSettings, MarketRecommendation and MarketReport. The owner wanted the old code removed and market intel rebuilt for the admin hub, so **ask before deleting the tables**. The 10 saved competitors were backed up, and he still has to answer the 4 questions in section 8.
- Do a full unused-code sweep: list each item, what it was for and where it is, then remove only after the owner approves.

### Task 5 — "Restrict browser features" header
- **Recommended: turn it on.** It lets only Miida's own pages use the camera, microphone, location, payment and USB; embedded third-party content can't. It also turns off motion sensors.
- Miida doesn't use the camera; it may use the **microphone** later for per-workspace voice AI, which still works for its own pages.
- How: `execute_api` `PUT /api/apps/{app_id}/security/headers` with `{ "restrict_browser_features": true }`. It applies to the published site immediately and can be reversed.
- The owner implied OK; confirm once, then do it.

---

## 8. Not implemented yet (owner's list)
- **Outside services:**
  - Cloudflare Turnstile keys: until they're added, signed-out visitors can't book an audit.
  - The SNS topic for SES bounces.
  - Telnyx texting: SMS replies, incoming texts, setup texts and the two-step text backup.
  - The Meta access token: for Meta marketing tools and instant-form lead import (`INSTANT_FORM_SYNC_ENABLED`).
  - Wix API key + `PAYMENT_FULFILLMENT_ENABLED`, on launch day.
- **Publishing the app.** Publishing also cleans up old function names and activates the Resend webhook.
- **Switch `ELDER_CLIENT_DATA_BYPASS` to false** before launch.
- **Sign-in:** two-step sign-in on; Google 2-step on the Elder Google accounts.
- **Make the GitHub repo private or delete it.**
- **Real-account role tests:** client, member, agent, manager, head and Elder, plus every public form and key action once for real (contact, data rights, audit booking, age check, account deletion, teammate invite).
- **A retention cleanup job** for records kept "as long as the law requires".
- **Market intel for the admin hub.** The owner still has to answer 4 questions:
  1. What should it track?
  2. Elder-only, or a grantable page?
  3. Keep the 10 saved competitors or start fresh?
  4. Daily scans or a "Scan now" button?
- **Schedule `reconcile-subscription-cancellations`** when billing launches.
- **Agent test account Calendar:** fixed by task 3C.
- **Later additions (owner's choice):**
  - **auto sign-out after inactivity** (added to the list this session);
  - failure alerts to Elders when sending, payments or bookings fail repeatedly;
  - CTA → workspace-fulfillment rehearsal automation;
  - email campaigns (Booms) for clients (hidden now, Elder-grantable for testing);
  - the Lead Finder as a product, maybe;
  - a microphone/voice model per workspace.
- **Data exports:** the owner will ask when he needs one.

---

## 9. Gotchas and lessons
- **Tables reject values not in their allowed list.** When adding a new status, scope or title, update the table's enum with `update_entity_schema`, sending the full schema; security rules are kept. Add the value to `tests/test-reservation-scopes.mjs` where relevant.
- **Updating the User schema:** fetch the live schema first with `GET /api/apps/{app_id}/entity-schemas/User` and send it back complete. Leaving out a field deletes it.
- **Test fakes and the stored-user re-read:** fake clients must have a User table containing the signed-in fake user, or the guard refuses with 503/401. The shared loader adds a fallback, but a test whose User table holds a *different* record for that ID will see the stored record win. That is correct; make the test's stored row match.
- **`run_command` and long jobs:** it times out at 60 seconds, so start the release check detached (`setsid nohup npm run release:check > /tmp/rc.log 2>&1 &`) and poll `/tmp/rc.log`.
- **Security scans:** run with `execute_api` `POST /api/apps/{app_id}/security/scan`, then poll `GET` after about 5–10 minutes. The scan is AI-based and finds new things on each pass, so read each finding critically. Several earlier claims were wrong (for example, "Spoke endpoints are dead").
- **The owner prefers plain language,** short explanations, and a plan before action.
