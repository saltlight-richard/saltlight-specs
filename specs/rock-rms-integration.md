# Rock RMS Integration — Backend Spec for Kenneth

**Replaces all prior versions. This is the single source of truth.**
Audience: Kenneth (backend engineer, new to Rock RMS). Last verified against the live test instance (`saltlight.rockcloud.com`, Rock 18.1.0.0) on **2026-06-02**.

### How to read this document

This spec is deliberately code-free. It tells you **what** each feature must do and **where** the data lives — never **how** to write it. You own all the code. Every feature follows the same shape so you can skim:

1. **User story** — one sentence: who wants what, and why.
2. **User journey** — the concrete step-by-step narrative. Where a feature has more than one path (e.g. ingestion has Track 1 and Track 2), each path gets its own subhead.
3. **Acceptance criteria** — testable, observable checkboxes. If you can't watch it happen in the database, a log, or Rock's UI, it isn't done.
4. **What the backend touches** — plain-English map of Lambda/handler, table + JSONB key, and endpoints. No logic.
5. **Reference material** — a table of every API call the feature makes, each with the OData query params and **two direct links**: the Swagger controller on the live install, and the named verified curl in the Test Environment doc.
6. **Gotchas** — the specific Rock traps that will bite you on this feature.

There are **five features**, built in this order: **1 (Connection) → 1b (Rock Form Setup) → 2 (Per-CareFlow config) → 3 (Ingestion, Rock → SaltLight) → 4 (Write-back, SaltLight → Rock)**. See the build order section for why.

### Breadcrumb / nav legend

Every feature section opens with a breadcrumb so you always know where you are:

> 🧭 Rock RMS Integration › Feature {ID} — {Name}

- **🧭** = you are here (top-of-section locator).
- **Track 1 / Track 2** subheads (Feature 3) = the two independent ingestion paths.
- **Section A / Section B** subheads (Feature 2) = inbound (Rock form → CareFlow) vs. outbound (SaltLight card → Rock) config.
- A checkbox `- [ ]` in Acceptance criteria = a thing a tester (Richard or you) must be able to observe.
- Code fences appear in this document **only** to show data shapes — an example API request/response body, or a stored JSONB config blob. They never contain logic. If you see a fence, it is data, not an instruction to copy code.

### One mental model before you start

SaltLight and Rock sync **both directions**, but each direction is owned by exactly one feature:

- **Rock → SaltLight** (someone shows up in Rock; SaltLight picks them up and starts follow-up) = **Feature 3, ingestion only**. It never writes to Rock.
- **SaltLight → Rock** (a card is captured in SaltLight; we push the guest into Rock and attribute them) = **Feature 4, write-back only**. It owns every create/update/tag/workflow call against Rock.

Keep those two lanes separate in your head and the rest of the spec falls into place.

> **Scope note on webhooks:** v1 ingestion is **polling only** (Feature 3, Track 1 + Track 2). There is **no webhook receiver in v1**. The `rock_webhook` source value is **reserved for a possible future (v2) inbound-webhook path** — its cards would be tagged `rock_webhook` — so the "never push a Rock-sourced card back to Rock" guards in Features 3 and 4 are already forward-compatible. But nothing in v1 creates a `rock_webhook`-sourced card. The `org_id` stored at connect time (= the Rock domain) is likewise **reserved for future webhook routing**; no v1 feature consumes it for ingestion.

## Rock in 90 seconds

Rock RMS is the church's system of record. SaltLight reads from it and writes to it over Rock's REST/OData API. Here is every Rock concept this spec uses, one line each, plain English:

- **Person** — a human record in Rock (the church's contact). Has `Id`, `FirstName`, `LastName`, `Email`, phone numbers, and a `Guid`. This is what we ingest (Track 1) and what we create/update on write-back.
- **WorkflowType** — a *template* for a process, e.g. "First Time Guest - SaltLight" (Id 24). Think of it as the definition of a form. Churches build their connect-card form as a WorkflowType.
- **Workflow instance** — one *filled-out copy* of a WorkflowType, e.g. a specific guest's submitted form. Has a `Status` ("Active", "Completed") and a `CreatedDateTime` set at submission. We ingest **Completed** instances (Track 2).
- **Attribute** — a field on a WorkflowType (e.g. "First Name", "Interest Area"). Identified by an integer `AttributeId` (e.g. FirstName = 8281 on the test install). The *definition* of a field, not its value.
- **AttributeValue** — the actual answer a guest gave for one Attribute on one Workflow instance. Keyed by `AttributeId` + `EntityId` (the workflow instance Id). This is where the guest's typed-in data lives.
- **DefinedValue** — a pre-set option in a dropdown list Rock manages centrally (e.g. ConnectionStatus = "Visitor", RecordStatus = "Active"). Each has an install-specific integer Id but a portable `Guid`.
- **Campus** — a physical church location (Main = Id 1, North = Id 2 on the test install). We cache the campus list at connect time and resolve campus Ids to names.
- **Tag** — a label you can stick on a Person in Rock (e.g. "Staff", Id 1). Applied via a TaggedItem row. Used for write-back attribution.
- **CampusPicker** — a field type whose AttributeValue stores the **Campus Id as a string** (e.g. `"1"`), not the campus name. Always resolve it through the campus cache to get a human name.

## Build & test order

Build and test the features **in this order**. Each one depends on the data or behavior the previous one establishes, so building out of order means testing against scaffolding that doesn't exist yet.

**1 → 1b → 2 → 3 → 4**

### Feature 1 — Rock Account Connection *(build first)*
**Why first:** nothing else can talk to Rock until an account is connected. Feature 1 establishes the credential (in Secrets Manager), the connection state in `account_metadata['rockrms']`, the campus cache, the `org_id` (the domain — stored as a routing key reserved for future webhook routing), and the initial poll cursors. It also runs the permissions probe so you find a bad/under-privileged key here, not three features later.
**Test:** connect a test account against `saltlight.rockcloud.com`; confirm `account_metadata['rockrms']` is populated with `connected: true`, the campus cache (Main=1, North=2), and both poll cursors. Confirm a key without People access surfaces the specific error.

### Feature 1b — Rock Form Setup (account-level field mapping) *(build second)*
**Why second:** Feature 1b is the bridge between "connected" and "useful." It uses the connection from Feature 1 to list the church's Rock forms (WorkflowTypes) and their fields (Attributes), then stores **one** field mapping per form (which AttributeId is first name, last name, email, phone, campus) at the account level in `account_metadata['rockrms']['mapped_forms']`. Feature 2 and Feature 3-Track-2 both read this mapping — so it must exist before either can be built or tested.
**Test:** map WorkflowType 24's attributes (8281–8285) and confirm `mapped_forms` holds the mapping. Verify the field-values dropdown resolves InterestArea (8286) options.

### Feature 2 — CareFlow Rock Configuration (Step 6) *(build third)*
**Why third:** Feature 2 lets an admin point a specific CareFlow at a **mapped** form (from 1b), set the **required triggering selection**, and optionally configure write-back (tags + workflow types). It can't reference a mapped form that doesn't exist, so 1b must land first. It produces `workflow_data['rockrms']`, which is the config Feature 3 (which form to poll, which selection triggers) and Feature 4 (which tag/workflow to write back) both consume.
**Test:** attach WorkflowType 24 to a CareFlow with the required triggering selection (InterestArea = Sunday Service) and a write-back tag (Id 1); confirm `workflow_data['rockrms']` persists the config.

### Feature 3 — Guest Intake: Rock → SaltLight (ingestion only) *(build fourth)*
**Why fourth:** ingestion is the first feature that actually *runs the loop*, and it needs all three prior features in place — a connected account (1), a form mapping to read values through (1b), and at least one CareFlow pointed at a form with its triggering selection set (2). It creates Connect Cards from Rock and manages the poll cursors. It does **not** write back. Ingestion is **polling only**: Track 1 reads new/updated People by `ModifiedDateTime`; Track 2 reads Completed Workflow instances by `CreatedDateTime`. There is no webhook path in v1.
**Test:** with the loaded test data (People 22–24, Workflows 8–10) and cursor `2026-06-02T12:11:00`, run the poller; confirm 3 Track-1 Connect Cards and 3 Track-2 Connect Cards appear, no duplicates, and both cursors advance.

### Feature 4 — Card Capture Push: SaltLight → Rock (write-back only) *(build last)*
**Why last:** write-back is the inverse direction and depends on everything before it — the connection (1), the mappings and CareFlow config that tell it which attributes and tag/workflow to use (1b + 2), and the source-tagging from ingestion (3) so it knows **not** to push Rock-originated cards back to Rock (the loop guard). Building it last means the "don't push back" guard can be tested against real Rock-sourced cards from Feature 3.
**Test:** scan/enter a non-Rock card; confirm the guest is created or matched in Rock, the configured tag is applied, a workflow instance is created with card fields populated, and the SaltLight person record stores `rockrms_person_id`. Confirm a Rock-sourced card (`rock_track1`/`rock_track2`) is never pushed back, and that a Rock outage queues the card as `pending` without blocking the CareFlow. (`rock_webhook` is a reserved future source — no v1 path produces it, but the guard already covers it.)

---

## Feature 1 — Rock Account Connection

> 🧭 Rock RMS Integration › Feature 1 — Rock Account Connection

### User story
As a church admin, I want to connect my Rock RMS account to SaltLight by entering my Rock domain and REST API key, so that guest data can flow automatically between both systems without any manual setup beyond this one step.

### User journey

This feature is a single path — there is no Track 1 / Track 2 split at connect time. It runs as an ordered sequence, and every step must succeed before the connection is marked live. Nothing else in the integration (Features 2, 3, 4) works until this completes.

1. The admin opens **Integrations → Rock RMS** in the SaltLight portal and enters two things: their **Rock domain** (e.g. `gracechurch.rockrms.com`) and their **Rock REST API key**.
2. SaltLight runs a **connectivity probe** — it confirms Rock is reachable at that domain. If the domain is wrong or Rock is not publicly accessible, the admin sees a "not reachable" message and the connection is not saved.
3. SaltLight runs a **permissions probe** — it calls `GET /api/People/GetCurrentPerson` as the key's user to confirm the key not only authenticates but has People read access. A key that authenticates but lacks a Rock security role fails here (HTTP 401) with a specific, actionable message (the message points the admin at Rock's security settings, not at SaltLight).
4. SaltLight **resolves three install-specific DefinedValue IDs** from their portable GUIDs (the GUIDs are identical on every Rock install; the integer IDs differ per install). It resolves: **Active RecordStatus**, **Visitor ConnectionStatus**, and **Mobile PhoneType**. Each is a separate `GET /api/DefinedValues` lookup that filters by the portable GUID and reads back the install's integer `Id`.
5. SaltLight **caches the active campus list** — it fetches all active campuses (`GET /api/Campuses?$filter=IsActive eq true`) and stores each campus's `Id` and `Name`. This cache is what every later feature uses to translate a campus reference into a name (and, for CampusPicker attribute values, to translate a stored string Campus Id back into a campus).
6. SaltLight **stores the credentials in AWS Secrets Manager** (never in the database). The secret path is namespaced per environment and per account.
7. SaltLight **writes connection state** to the account record: connected flag, church name, domain, the `org_id` (set equal to the domain — reserved for future webhook routing; no v1 feature consumes it), the three resolved DefinedValue IDs, and the campus cache.
8. SaltLight **initializes both poll cursors and `last_polled`** to the connection timestamp (`connected_at`). This is the "start from now, do not backfill history" rule — the first poll only looks at records changed after the moment of connection. Critically, `connected_at` must be expressed on **Rock's clock** (the install's local server time), not UTC (see Gotchas).
9. The connection is now **live**. The admin sees a success state, and the poll Lambda will pick this account up on its next scheduled run.

**Worked example.** Pastor Tom enters `gracechurch.rockrms.com` and his key. SaltLight reaches Rock (step 2), confirms People access (step 3), resolves Active = 3 / Visitor = 66 / Mobile = 12 for his install (step 4), caches "Main Campus" (Id 1) and "North Campus" (Id 2) (step 5), stores the key in Secrets Manager (step 6), writes the connection state with `org_id = gracechurch.rockrms.com` (step 7), and sets both cursors to now-on-Rock's-clock (step 8). Grace Church is connected.

**Failure example.** Tom's key authenticates but his Rock user is not in a security group with People read access. Step 3 returns HTTP 401. SaltLight tells him: "Your API key is valid, but the user account it belongs to lacks permissions. In Rock, add that user to a security group with People read access." Nothing is saved; he fixes it in Rock and retries.

### Acceptance criteria
- [ ] Connection succeeds **only if** Rock is reachable at the domain **AND** the key's user has People read access; if either fails, no connection state is written and no credentials are stored.
- [ ] An unreachable domain returns a distinct "Rock is not reachable at that URL — verify the domain and that Rock is publicly accessible" message; a valid-key-but-no-permission case (HTTP 401 on `GET /api/People/GetCurrentPerson`) returns a distinct message that points the admin to Rock's security settings.
- [ ] On success, the account's connection state contains: `connected: true`, `domain`, `org_id` equal to the domain, `active_status_id`, `visitor_id`, `mobile_id`, and a `campuses` array of `{id, name}` for every active campus.
- [ ] The three resolved IDs are the **install-specific integers** obtained by resolving the portable GUIDs at connect time via `GET /api/DefinedValues?$filter=Guid eq guid'<GUID>'` — never hardcoded values.
- [ ] The REST API key is stored in AWS Secrets Manager (per-env, per-account path) and is **not** written to the database in any form.
- [ ] `poll_cursor_track1`, `poll_cursor_track2`, and `last_polled` are all initialized to `connected_at` at connect time, and `connected_at` is aligned to Rock's local server clock (not UTC) — stored as a plain `YYYY-MM-DDThh:mm:ss` string with **no trailing `Z` and no offset**. On the test install (`saltlight.rockcloud.com`) the offset is ~7 hours behind UTC; verify the live install's offset.
- [ ] After a successful connect, the next scheduled poll run (Feature 3) is able to find this account and read all of the above without any further admin action.

### What the backend touches
- **Handler (orchestrator):** the Rock connect handler in `orchestrator/src/rockrms.py` (the entry point for the connect request from the portal) which calls into the shared integration's `create_connection()`.
- **Shared integration:** `shared/integrations/rockrms.py` — performs the connectivity probe, the permissions probe, the three GUID→Id resolutions, the campus fetch, and then writes connection state via the existing `set_connection()` path.
- **Credentials store:** AWS Secrets Manager at `saltlight/rockrms/tokens/{env}/account-{account_id}` — the key lives here only.
- **Connection state:** `public.account.account_metadata['rockrms']` (JSONB). Keys written/initialized here: `connected`, `church_name`, `connected_at`, `domain`, `org_id`, `active_status_id`, `visitor_id`, `mobile_id`, `campuses`, `poll_cursor_track1`, `poll_cursor_track2`, `last_polled`, `poll_schedule`.
- **Reserved-for-future routing:** `org_id` (= domain) is stored at connect time as a forward-compatible routing key. It is **not consumed by any v1 feature** — webhooks are out of scope for v1 (ingestion is polling-only: Track 1 + Track 2 in Feature 3). If a Rock webhook receiver ships in a future version, `org_id` is the key it would use to route an inbound webhook back to this account; until then it is written but unused. Do not gate any v1 path on it.

### Reference material

| What | Endpoint (+ key query params) | Method | Direct links |
|---|---|---|---|
| Permissions probe — confirm key has People read access (200 = OK, 401 = missing security role) | `/api/People/GetCurrentPerson` | GET | [Swagger › People](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Connection: permissions probe"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| Resolve a portable GUID to this install's integer DefinedValue Id (run once each for Active RecordStatus, Visitor ConnectionStatus, Mobile PhoneType) | `/api/DefinedValues?$filter=Guid eq guid'<GUID>'&$select=Id` | GET | [Swagger › DefinedValues](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Connection: resolve DefinedValue GUID"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| Cache active campuses (bare-array response) | `/api/Campuses?$filter=IsActive eq true&$select=Id,Guid,Name,ShortCode` | GET | [Swagger › Campuses](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Campuses list"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| General: Rock REST API + OData conventions | OData query options: `$filter`, `$expand`, `$select`, `$orderby`, `$top` | — | [Rock REST API overview](https://community.rockrms.com/developer/rock-api) · [Rock developer hub](https://community.rockrms.com/developer) · [Canonical spec hub](https://saltlight-richard.github.io/saltlight-specs/specs/rock-rms.html) |

> Live API explorer: every endpoint above is browsable on the install's Swagger UI at `https://saltlight.rockcloud.com/api/docs` (append the controller name — People, DefinedValues, Campuses — to jump to that controller). Verified curls for each call live in the [Rock RMS Test Environment doc](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit).

**Portable GUIDs to resolve (identical on every Rock install; the integer Ids differ per install):**

```json
{
  "active_record_status_guid":      "618F906C-C33D-4FA3-8AEF-E58CB7B63F1E",
  "visitor_connection_status_guid": "B91BA046-BC1E-400C-B85D-638C1F4E0CE2",
  "mobile_phone_type_guid":         "407E7E45-7B2E-4FCD-9605-ECB1339F2453"
}
```

**Resolved IDs on the test install** (`saltlight.rockcloud.com`, Rock 18.1.0.0 — for verification only, never hardcode):

```json
{ "active_status_id": 3, "visitor_id": 66, "mobile_id": 12 }
```

**Campus list response shape** — note this is a **bare array**, not wrapped in `{"value": [...]}`:

```json
[
  { "Id": 1, "Guid": "...", "Name": "Main Campus", "ShortCode": "MAIN" },
  { "Id": 2, "Guid": "...", "Name": "North Campus", "ShortCode": "NORTH" }
]
```

**Stored connection-state shape** written to `account_metadata['rockrms']` (`connected_at` is Rock-local time, ~7h behind UTC on the test install — no trailing `Z`, no offset):

```json
{
  "connected": true,
  "church_name": "Grace Church",
  "connected_at": "2026-06-02T12:11:00",
  "domain": "gracechurch.rockrms.com",
  "org_id": "gracechurch.rockrms.com",
  "active_status_id": 3,
  "visitor_id": 66,
  "mobile_id": 12,
  "campuses": [
    { "id": 1, "name": "Main Campus" },
    { "id": 2, "name": "North Campus" }
  ],
  "poll_cursor_track1": "2026-06-02T12:11:00",
  "poll_cursor_track2": "2026-06-02T12:11:00",
  "last_polled": "2026-06-02T12:11:00",
  "poll_schedule": {
    "high_frequency_days": [5, 6],
    "high_frequency_start": "15:00",
    "high_frequency_end": "21:00",
    "low_frequency_interval_hours": 12,
    "timezone": "America/Chicago"
  }
}
```

The `poll_schedule` block (initialized here, consumed by Feature 3 and shown in the Storage map) means: every 5 min on Saturday/Sunday (`high_frequency_days: [5, 6]`) during the service window; otherwise twice daily (`low_frequency_interval_hours: 12`). EventBridge fires every 5 min and a should-poll gate decides whether to actually poll. Per-church configurable.

### Gotchas
- **Rock returns timestamps in LOCAL server time, not UTC.** The test install runs ~7 hours behind UTC (a record created at 19:11 UTC comes back stamped 12:11). `connected_at` (and therefore both initialized cursors and `last_polled`) must be written on Rock's clock as a plain `YYYY-MM-DDThh:mm:ss` string — **no trailing `Z`, no offset**. If you store a UTC `connected_at` against a Rock that reports local time, the very first poll filter lands in the future and silently returns zero records — the connection looks healthy but never ingests anything. Confirm the install's offset and align the cursor to it.
- **GUID filters require the `guid'...'` prefix.** `$filter=Guid eq guid'618F906C-...'` works; a bare quoted string (`'618F906C-...'`) does not. This applies to all three DefinedValue resolutions.
- **All Rock list endpoints return bare arrays**, not `{"value": [...]}`. This includes `/api/DefinedValues` and `/api/Campuses` used here (and, contrary to the old spec, `/api/People`, `/api/Workflows`, `/api/AttributeValues`, `/api/WorkflowTypes`, and `/api/Tags` too — verified on Rock 18.1.0.0). Read the response as a list directly — treating it as an object and calling `.get('value', ...)` will fail.
- **Each DefinedValue GUID must be resolved separately.** There is no single call that returns all three. Resolve Active RecordStatus, Visitor ConnectionStatus, and Mobile PhoneType independently, and fail the connect if any one cannot be resolved (a missing GUID means the install is non-standard and downstream person creation / phone-matching would be wrong).
- **A 401 on the permissions probe is almost always a missing Rock security role, not a bad key.** The key can authenticate and still be denied People access. Surface a message that sends the admin to Rock's security settings (add the user to a group with People read access) — do not tell them the key is invalid.
- **Single-record GETs ignore `$select`** and return the full object. `GetCurrentPerson` and `/api/DefinedValues/{id}` will return everything regardless of `$select` — read fields defensively by name rather than assuming a trimmed response shape.
- **`$select` silently strips `PhoneNumbers` from `$expand=PhoneNumbers`** — never combine the two on `/api/People`. (Relevant later, at poll time; noted here so it is not introduced as a habit.)
- **`$expand=Campus` on People is silently ignored**, and `PrimaryCampusId` is often null on this install (campus actually lives in the Family Group model). That is why the campus **cache** built here is the source of truth for campus name resolution in later features — do not plan to expand campus inline at poll time.
- **CampusPicker attribute values store the Campus Id as a string** (e.g. `"1"`), not the name. The cache stored here is what later features use to turn that string Id back into a campus — another reason this step must succeed at connect time.
- **OData datetime filters must use `datetime'YYYY-MM-DDThh:mm:ss'`.** Although this connect flow does not filter by datetime, the cursors initialized here are consumed by Feature 3 with that exact syntax — a bare ISO string there returns HTTP 400. Store the cursor in plain `YYYY-MM-DDThh:mm:ss` form (no trailing `Z`, no offset) so Feature 3 can wrap it directly.
- **Store the key only in Secrets Manager.** The connection state in `account_metadata['rockrms']` holds non-secret config (IDs, campuses, cursors, schedule, domain). The API key must never be written into that JSONB or anywhere in the database.
- **`org_id` is reserved-for-future, not load-bearing in v1.** It is written at connect time (= domain) so that a future v2 webhook receiver could route inbound Rock webhooks back to this account, but **webhooks are out of scope for v1** — ingestion is polling-only (Track 1 + Track 2, Feature 3). No v1 feature reads `org_id` and no `rock_webhook`-sourced card is created in v1. Do not gate any connect step or downstream poll on it, and do not describe a "webhook fallback path" as if it ships now.
- **This is a hard prerequisite.** Features 2, 3, and 4 all read the IDs, campus cache, cursors, and poll schedule written here. If connect is allowed to "partially succeed" (e.g. campuses fetched but a GUID unresolved), downstream features will misbehave silently — treat the whole sequence as all-or-nothing.
## Feature 1b — Rock Form Setup (account-level field mapping)

> 🧭 Rock RMS Integration › Feature 1b — Rock Form Setup (account-level field mapping)

### User story
As a church admin, I want to tell SaltLight which Rock forms to watch and which form field holds each guest detail (first name, last name, email, phone, campus), so that every CareFlow can read guests out of those forms correctly without me re-mapping the fields each time.

### User journey
This is a **one-time, account-level** setup. The admin does it once per Rock form. The resulting mapping is saved at the account level and is reused by **every** CareFlow that references that form — it is **not** configured per-CareFlow.

1. After the Rock account is connected (Feature 1 — Rock Account Connection), the admin opens the "Rock Form Setup" screen in the SaltLight portal.
2. SaltLight shows a list of the church's active Rock forms. (Behind the scenes this is Rock's list of active WorkflowTypes — e.g. "First Time Guest - SaltLight".) The admin picks the form(s) SaltLight should watch.
3. For each form the admin selects, SaltLight loads that form's fields (Rock attributes) — each with a human label like "First Name", "Email Address", "Which campus did you visit?".
4. **SaltLight auto-suggests a mapping** for the five SaltLight fields — first name, last name, email, phone, campus. It does this by keyword string-matching the field labels/keys it already loaded in step 3. **No extra Rock API call is made for the suggestion** — it is pure string matching over data already in hand. For example, a field labeled "First Name" or keyed `FirstName` is suggested as `first_name`; "Email", "Email Address", or `Email` → `email`; "Mobile", "Cell", "Phone" → `phone`; "Campus", "Location", "Site" → `campus`.
5. The admin reviews the suggestions and **corrects any wrong guesses** using a dropdown per SaltLight field — the dropdown lists every field on that Rock form. A field with no good match can be left unmapped.
6. The admin saves. SaltLight records, for that form, the Rock form id and name plus the chosen Rock field id for each of the five SaltLight fields.
7. The mapping is now live at the account level. When Feature 3 — Guest Intake: Rock → SaltLight later ingests a completed submission of this form, it reads each value by looking up the saved field ids — and any CareFlow pointed at this form (Feature 2 — CareFlow Rock Configuration) reuses the same mapping.

Re-running setup for a form the admin already mapped should re-load the form's fields, pre-fill the dropdowns with the previously saved choices, and overwrite the existing entry on save (not create a duplicate).

### Acceptance criteria
- [ ] The form picker lists only **active** Rock forms — i.e. `/api/WorkflowTypes` filtered by `IsActive eq true` — each showing its Rock name, so the admin recognizes their own forms.
- [ ] Selecting a form loads that form's fields with their human-readable labels; an empty form (no fields) shows an empty mapping rather than an error.
- [ ] On loading a form's fields, SaltLight pre-fills a suggested mapping for first name, last name, email, phone, and campus **without making any additional Rock API call** for the suggestion (string-match only over the labels/keys already returned by the field-list call).
- [ ] The admin can override any suggested mapping via a per-field dropdown listing all of that form's fields, and can leave a SaltLight field unmapped.
- [ ] Saving writes one entry per mapped form into `account_metadata['rockrms']['mapped_forms']`, each entry containing `workflow_type_id`, `workflow_type_name`, and an `attribute_ids` object with the chosen Rock field id for `first_name`, `last_name`, `email`, `phone`, `campus`.
- [ ] Re-saving a form that is already mapped updates the existing entry in place (matched by `workflow_type_id`) — it does not create a second entry for the same form.
- [ ] The same saved mapping is used by every CareFlow that references the form; the admin never re-maps fields inside a CareFlow (Feature 2 only points at a mapped form, it does not store field ids).
- [ ] Saved Rock field ids are stored as integers (Rock AttributeIds), matching what the Rock API returns.
- [ ] **Live test-data check:** mapping WorkflowType `24` ("First Time Guest - SaltLight") on the test install and accepting the suggested fields persists `attribute_ids` of exactly `{first_name:8281, last_name:8282, email:8283, phone:8284, campus:8285}`. (`InterestArea` `8286` is Feature 2's trigger field, NOT one of the five mapped core fields.)

### What the backend touches
- **Handler / Lambda:** the Rock forms endpoints in the orchestrator Rock handler (`orchestrator/src/rockrms.py`) — these serve the portal's form picker, the per-form field list, and (where a field is a dropdown/single-select) its option values. The same handler dispatches all three based on which query params are present.
- **Integration layer:** the Rock methods in `shared/integrations/rockrms.py` that fetch active WorkflowTypes, fetch a WorkflowType's attributes, and (optionally) fetch a single attribute's option values. These are read-only calls to Rock for this feature.
- **Suggestion logic:** runs **in SaltLight** (portal and/or handler), purely over the field labels/keys returned by the field-list call. It does not call Rock.
- **Storage (read + write):** `public.account.account_metadata` JSONB, under the `rockrms` key, in the `mapped_forms` array. Feature 1b is the only feature that writes `mapped_forms`. Feature 2 reads it (to offer a form picker) and Feature 3 reads it (to interpret submissions). This setup writes **nothing** to Rock — it is account-level configuration only.

### Reference material

The Swagger explorer at `https://saltlight.rockcloud.com/api/docs` is a single live page on the test install; reach a specific controller (WorkflowTypes, Attributes, AttributeQualifiers, DefinedTypes, DefinedValues) by selecting it in the explorer's controller list. The "Test Env" links point at named, already-verified curls in the SaltLight Rock RMS Test Environment doc.

| What | Endpoint (+ key query params) | Method | Direct links |
|---|---|---|---|
| List the church's active forms (form picker) | `/api/WorkflowTypes?$filter=IsActive eq true&$select=Id,Guid,Name&$orderby=Name asc` | GET | [Swagger › WorkflowTypes](https://saltlight.rockcloud.com/api/docs) · [Test Env › "WorkflowTypes list"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| List a form's fields (for mapping + suggestion) | `/api/Attributes?$filter=EntityTypeId eq 113 and EntityTypeQualifierColumn eq 'WorkflowTypeId' and EntityTypeQualifierValue eq '<workflow_type_id>'&$select=Id,Key,Name,FieldTypeId` | GET | [Swagger › Attributes](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Form fields (Attributes)"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| List option values for a dropdown field (DefinedValue picker path) | `/api/AttributeQualifiers?$filter=AttributeId eq <attr_id> and Key eq 'definedtypeguid'&$select=Value` then `/api/DefinedTypes?$filter=Guid eq guid'<guid>'&$select=Id` then `/api/DefinedValues?$filter=DefinedTypeId eq <id>&$select=Id,Value&$orderby=Order asc` | GET | [Swagger › AttributeQualifiers](https://saltlight.rockcloud.com/api/docs) · [Swagger › DefinedValues](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Field values (Value List)"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| List option values for a "Value List" field (pipe-delimited path) | `/api/AttributeQualifiers?$filter=AttributeId eq <attr_id> and Key eq 'values'&$select=Value` (parse `Label,value` pairs split on `\|`) | GET | [Swagger › AttributeQualifiers](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Field values (Value List)"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| API explorer (live Swagger on the test install) | `https://saltlight.rockcloud.com/api/docs` | — | [Rock REST API explorer](https://saltlight.rockcloud.com/api/docs) |
| Rock REST/OData conventions | $filter, $select, $orderby on `/api/{Entity}` | — | [Rock REST API overview](https://community.rockrms.com/developer/rock-api) · [Rock developer hub](https://community.rockrms.com/developer) |
| Canonical spec hub | — | — | [Rock RMS spec hub](https://saltlight-richard.github.io/saltlight-specs/specs/rock-rms.html) |

**Test data to drive a real mapping (already live, per Test Env doc):** WorkflowType `24` = "First Time Guest - SaltLight", with AttributeIds FirstName `8281`, LastName `8282`, Email `8283`, Phone `8284`, Campus `8285`, InterestArea `8286`. A correct save of form 24 should produce `attribute_ids` of `{first_name:8281, last_name:8282, email:8283, phone:8284, campus:8285}` — `InterestArea` `8286` is Feature 2's trigger field and is deliberately not one of the five mapped core fields. (Campuses on this install: `1` = Main, `2` = North.)

**Stored config shape** — written to `account_metadata['rockrms']['mapped_forms']` (an array; one entry per mapped form):

```json
{
  "mapped_forms": [
    {
      "workflow_type_id": 24,
      "workflow_type_name": "First Time Guest - SaltLight",
      "attribute_ids": {
        "first_name": 8281,
        "last_name": 8282,
        "email": 8283,
        "phone": 8284,
        "campus": 8285
      }
    }
  ]
}
```

**Example field-list response shape** (what the suggestion logic string-matches against — note this is a bare array, not `{"value":[...]}`):

```json
[
  { "Id": 8281, "Key": "FirstName", "Name": "First Name", "FieldTypeId": 1 },
  { "Id": 8283, "Key": "Email", "Name": "Email Address", "FieldTypeId": 71 },
  { "Id": 8285, "Key": "Campus", "Name": "Which campus?", "FieldTypeId": 53 }
]
```

### Gotchas
- **Every Rock list endpoint returns a BARE ARRAY**, not `{"value":[...]}`. This is verified live on Rock 18.1.0.0 across `/api/WorkflowTypes`, `/api/Attributes`, `/api/AttributeQualifiers`, `/api/DefinedTypes`, `/api/DefinedValues` — and across `/api/People` and `/api/Workflows` too. Read every list directly; do **not** reach for a `value` wrapper or you'll get a "list has no attribute" failure. (An older test-env note claimed People/Workflows return `{"value":[...]}` — that claim is wrong; treat all list reads as bare arrays, no exceptions.)
- **The auto-suggestion makes NO Rock API call.** It is keyword string-matching over the field labels/keys already returned by the field-list call. Don't add a "smarter" Rock lookup to improve guesses — admin correction is the safety net, and the match list (synonyms per field) is the only thing that needs tuning.
- **A form's fields are Rock Attributes scoped by qualifier, not generic attributes.** You must filter `/api/Attributes` by `EntityTypeId eq 113` AND `EntityTypeQualifierColumn eq 'WorkflowTypeId'` AND `EntityTypeQualifierValue eq '<the form id>'`. **`EntityTypeId 113` is the Workflow entity type — verified in the Test Environment doc, not a magic number.** Without all three qualifiers you get attributes from unrelated entities.
- **GUID filters need the `guid'...'` prefix.** When the option-values path resolves a DefinedType by Guid, the filter must be `Guid eq guid'XXXXXXXX-...'`, not a bare quoted string — a bare string returns no rows.
- **The campus field maps to an AttributeId here, but its stored submission value is a STRING Campus Id, not a name.** A Rock CampusPicker stores `"1"` (a string), not "Main". Feature 1b only saves the field id; resolving the string Campus Id to a campus name happens downstream at ingestion (Feature 3) against the campus cache. Don't try to resolve campus names at mapping time.
- **Single-record GETs ignore `$select`.** If you ever fetch a single WorkflowType or Attribute by id while building this screen, Rock returns the full object regardless of `$select` — read fields defensively by name.
- **Store the Rock field ids as integers.** Rock's AttributeIds come back as integers (e.g. `8281`); persist them as integers in `attribute_ids` so Feature 3's value lookups match without type coercion.
- **This mapping is account-level and shared.** Do not let it leak into `workflow_data['rockrms']` (that's Feature 2's per-CareFlow config, which only references a mapped form). The old version of this spec stored field ids per-CareFlow — that is superseded. One mapping per form, reused everywhere.
## Feature 2 — CareFlow Rock Configuration (Step 6)

> 🧭 Rock RMS Integration › Feature 2 — CareFlow Rock Configuration (Step 6)

This is **per-CareFlow** configuration. It is a thin layer on top of Feature 1b: Feature 1b maps a Rock form's fields to SaltLight fields **once, at the account level**; Feature 2 connects a **specific CareFlow** to Rock in two directions. There is **no field mapping in this feature** — Step 6 never asks "which Rock field is the email?" That lives in Feature 1b. Step 6 has two parts: **Section A (inbound)** — which mapped form and which selection on it triggers this CareFlow (both required) — and **Section B (outbound)** — when a card originates in SaltLight, which Rock workflow to write the guest into, plus optional tags (always shown, optional).

### User story
As a church admin, I want to attach a mapped Rock form to a specific CareFlow **and specify which selection on that form triggers it** — and optionally choose what to write back to Rock — so that a guest who makes that selection (e.g. checks "Accepted Jesus") lands in the right CareFlow, and my Rock database stays correctly attributed, without any field-by-field setup here.

### User journey

The admin is in the CareFlow setup wizard and reaches **Step 6 ("Rock Connection")**. It has two clearly-labeled sections, one per direction data can flow — the same split the Planning Center wizard uses ("PCO Form Submissions" vs "Physical Cards & Manual Entry"):

- **Section A — Inbound:** a Rock form submission starts this CareFlow. Both questions here are **required** (the selection is how a submission routes to the right CareFlow).
- **Section B — Outbound:** a connect card that originates in SaltLight (card scan or manual entry) is pushed into Rock. Always shown, but optional.

#### Section A — Inbound (a Rock form submission starts this CareFlow)

**A1 — Which mapped form? (required to enable Rock for this CareFlow)**
1. The screen loads the church's **mapped forms** — the Rock forms that were already mapped at the account level in Feature 1b. (These are read from `account_metadata['rockrms']['mapped_forms']`; the underlying list of available Rock forms comes from the WorkflowTypes lookup endpoint.)
2. The admin picks one mapped form (e.g. "First Time Guest - SaltLight", WorkflowType Id 24 on the test instance). This binds the CareFlow to that form's WorkflowType.
3. If the admin picks nothing and saves, the CareFlow simply has no Rock connection — it works normally without Rock. A church with no Rock integration leaves Step 6 blank.

**A2 — Which selection triggers this CareFlow? (required)**
1. A connect-card form usually asks the guest to make a selection — e.g. "How can we best connect with you?" with options like *Connect Groups, Serve Teams, Baptism, Accepted Jesus*. **This question tells SaltLight which selection fires this CareFlow.** It is required: without it, the system can't know that an "Accepted Jesus" submission belongs to the "Accepted Jesus" CareFlow, and the CareFlow would fire on *every* submission of that form regardless of what the guest picked — which is wrong.
2. The admin chooses the **trigger field** — the question on the form that carries the selection (e.g. "How can we best connect with you?"). Its options come from the form-fields lookup for that WorkflowType.
3. SaltLight reads that field's Rock **`FieldTypeId`** (already returned by the form-fields lookup) and **auto-detects whether it is single-select** (one value per submission, like a dropdown) **or multi-select** (a guest can check several, like a checkbox group). It shows this as a **pre-filled, editable** choice — the same "Checkbox or Dropdown?" question the Planning Center wizard asks, but pre-answered from Rock's field type — so the admin can confirm or override it. The resolved choice is saved as **`trigger_match_mode`**: `"equals"` for single-select, `"any_of"` for multi-select.
4. SaltLight populates that field's **real options**, and the admin **checks the selection(s) that should fire this CareFlow** (e.g. "Accepted Jesus") — chosen from real values, never free text. These are saved as `trigger_values`. Example: trigger field = "Interest Area" (AttributeId 8286 on the test instance), selection = ["Sunday Service"].
5. At ingestion (Feature 3), a submission becomes a Connect Card for this CareFlow **only when its selection matches** — **exact match** when `trigger_match_mode` is `"equals"`, or **"any checked value is in the selected set"** when it is `"any_of"`.

#### Section B — Outbound (a SaltLight-originated card is written into Rock)

When a connect card **originates in SaltLight** — scanned with CardCapture or entered by hand — SaltLight can push that guest into Rock. This section is **always shown but optional**: choose what should happen in Rock, or leave it empty to push nothing. (The push itself is performed by Feature 4 — Section B only stores the choices.)

**B1 — Which Rock workflow should we write the guest into?**
1. A **multi-select** of Rock WorkflowType ids (options from the WorkflowTypes lookup endpoint). For each one chosen, Feature 4 creates a Workflow instance of that type for the guest and fills in their fields. Saved as `writeback_workflow_type_ids`. Example: [24].

**B2 — Which tags should we apply?**
1. A **multi-select** of Rock person Tag ids (options from the Tags lookup endpoint). Saved as `writeback_tag_ids`. Example: [1] (Tag Id 1 = "Staff" on the test instance).

B1 and B2 are independent and both optional. If neither is chosen, no write-back happens for cards captured into this CareFlow.

> **Rock scope note:** SaltLight writes follow-up into Rock **Workflows** — it creates a Workflow instance of the chosen WorkflowType (verified live: `POST /api/Workflows` returns the new Id, then `POST /api/AttributeValues` fills the guest's fields, and `POST /api/TaggedItems` with `EntityTypeId 15` applies tags). It does **not** write to Rock **Connection Requests** or **Group** enrollment. If a church models its follow-up as Connections or Groups instead, that is out of scope for v1 (a possible v2 add, not a redesign) — the same assumption the PCO integration makes.

**Saving**
1. On Save, SaltLight writes the answers into `workflow_data['rockrms']` on this CareFlow row as a single-element array. No Rock write happens at save time — Step 6 only ever *reads* from Rock to populate dropdowns.
2. **Section A** (form + triggering selection) is **required** once Rock is enabled for this CareFlow — they are saved together as the inbound trigger. **Section B** (write-back) is **independent and optional** — a Rock-connected CareFlow may have outbound write-back or not, but it always has an inbound form and triggering selection.

### Acceptance criteria
- [ ] Step 6 lists only forms that were already mapped in Feature 1b (read from `account_metadata['rockrms']['mapped_forms']`); it does not let the admin map fields here.
- [ ] Selecting a mapped form and saving stores a single-element array under `workflow_data['rockrms']` whose element carries the form's WorkflowType id in `workflow_type_id`.
- [ ] The trigger attribute dropdown is populated from the selected form's fields (the Attributes lookup); the trigger-values multi-select is populated from that attribute's real option values (not free text) when the field is a dropdown / defined-value / value-list type.
- [ ] Saving a Rock-connected CareFlow stores `trigger_attribute_id`, a `trigger_match_mode` of `"equals"` or `"any_of"`, and a `trigger_values` array of one or more selections — **all required** once a form is chosen.
- [ ] A CareFlow that has a form selected but **no triggering selection cannot be saved** — the API rejects it with a clear "choose which selection on this form triggers this CareFlow" message — so a CareFlow never fires on every submission by accident.
- [ ] When a trigger attribute is chosen, SaltLight auto-detects single- vs multi-select from the field's `FieldTypeId`, displays it pre-filled and editable (the "Checkbox or Dropdown?" choice), and saves the resolved value as `trigger_match_mode`; the admin can override the auto-detection before saving.
- [ ] Section B's workflow selector (B1) is a multi-select populated from the WorkflowTypes lookup; saving stores an array of WorkflowType ids in `writeback_workflow_type_ids`.
- [ ] Section B's tag selector (B2) is a multi-select populated from the Tags lookup (Person tags only, `EntityTypeId eq 15`); saving stores an array of tag ids in `writeback_tag_ids`.
- [ ] Section B (outbound write-back) is **always shown but optional**; leaving both B1 and B2 empty saves cleanly with no write-back. SaltLight writes follow-up into Rock **Workflows** (creating a Workflow instance of the chosen type) — not Connection Requests or Group enrollment.
- [ ] Section A (write-back aside) is required once Rock is enabled: a fully blank Step 6 (no form chosen) leaves the CareFlow with no Rock connection; once a form is chosen, both the form and a triggering selection are required to save.
- [ ] If a campus-type field is chosen as the trigger attribute, the multi-select displays campus names but persists each campus **Id as a string** (e.g. `"1"`); a campus field is single-select (`trigger_match_mode: "equals"`), so Feature 3's exact-string comparison matches what Rock stores.
- [ ] Re-opening Step 6 after a save shows the previously selected form, triggering selection, write-back workflows, and tags exactly as stored.

### What the backend touches
- **CareFlow wizard (Step 6) save path** writes the config; no new Lambda handler is required for the save itself if the wizard persists through the existing CareFlow/workflow update path. The Rock-specific work here is **read-only lookups** to populate the Step 6 dropdowns.
- **Lookup handlers in `orchestrator/src/rockrms.py`** serve the dropdown data the portal needs:
  - the **forms** handler (`GET /integrations/rockrms/forms`) backs Section A (A1 + A2) and Section B's B1 — with no params it returns the WorkflowTypes list (used by A1's form picker and B1's write-back workflow picker); with `form_id` it returns that form's fields (A2); with `form_id` + `form_field_id` it returns that field's option values (A2),
  - the **tags** handler (`GET /integrations/rockrms/tags`, Person tags) backs Section B's B2.
  - Each lookup handler resolves the account, opens a `RockRMS` integration instance, and returns a bare list of `{id, name/label}` objects for the UI.
- **Table + JSONB key written:** `public.workflow` → `workflow_data['rockrms']` (a JSONB **array**; v1 uses a single element). The mapped-forms source it reads against is `public.account` → `account_metadata['rockrms']['mapped_forms']` (owned by Feature 1b — Feature 2 only reads it). Campus-name display for a campus trigger attribute is resolved against the campus cache at `account_metadata['rockrms']['campuses']` (written at connect time, Feature 1).
- **No Rock write** occurs in this feature. Applying tags and launching workflows in Rock is Feature 4 (write-back), which consumes the `writeback_tag_ids` / `writeback_workflow_type_ids` stored here. (Note for Feature 4: its `POST /api/TaggedItems` for a person tag must include `EntityTypeId: 15`, or Rock returns HTTP 500.)

### Reference material

Every API row below gives **two direct links**: the Swagger controller view and the exact named curl in the Test Environment doc.

| What | Endpoint (+ key query params) | Method | Direct links |
|---|---|---|---|
| List active Rock forms (Section A · A1 — "which form?") | `/api/WorkflowTypes?$filter=IsActive eq true&$select=Id,Guid,Name&$orderby=Name asc` | GET | [Swagger › WorkflowTypes](https://saltlight.rockcloud.com/api/docs) · [Test Env › "WorkflowTypes list"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| List a form's fields (Section A · A2 — pick the trigger field) | `/api/Attributes?$filter=EntityTypeId eq 113 and EntityTypeQualifierColumn eq 'WorkflowTypeId' and EntityTypeQualifierValue eq '<workflow_type_id>'&$select=Id,Key,Name,FieldTypeId` (EntityTypeId 113 = the Workflow entity type — verified in the Test Env doc) | GET | [Swagger › Attributes](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Form fields (Attributes)"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| List a trigger attribute's option values — DefinedValue picker | `/api/AttributeQualifiers?$filter=AttributeId eq <attr_id> and Key eq 'definedtypeguid'&$select=Value`, then `/api/DefinedTypes?$filter=Guid eq guid'<guid>'&$select=Id`, then `/api/DefinedValues?$filter=DefinedTypeId eq <id>&$select=Id,Value&$orderby=Order asc` | GET | [Swagger › AttributeQualifiers](https://saltlight.rockcloud.com/api/docs) · [Swagger › DefinedValues](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Field values (DefinedValue)"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| List a trigger attribute's option values — Value List field | `/api/AttributeQualifiers?$filter=AttributeId eq <attr_id> and Key eq 'values'&$select=Value` (returns a pipe-delimited string of `Label,value` pairs) | GET | [Swagger › AttributeQualifiers](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Field values (Value List)"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| List person Tags (Section B · B2 — write-back tags multi-select) | `/api/Tags?$filter=EntityTypeId eq 15&$select=Id,Guid,Name&$orderby=Name asc` (EntityTypeId 15 = Person) | GET | [Swagger › Tags](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Tags list"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| List WorkflowTypes for write-back (Section B · B1 — workflow types multi-select) | `/api/WorkflowTypes?$filter=IsActive eq true&$select=Id,Guid,Name&$orderby=Name asc` (same WorkflowTypes lookup as A1) | GET | [Swagger › WorkflowTypes](https://saltlight.rockcloud.com/api/docs) · [Test Env › "WorkflowTypes list"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| Resolve campus Id → name (only if a campus field is the trigger attribute) | Read from the connect-time cache `account_metadata['rockrms']['campuses']` (source: `/api/Campuses?$filter=IsActive eq true&$select=Id,Guid,Name,ShortCode`) | GET | [Swagger › Campuses](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Campuses list"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |

General references: [Rock REST API overview](https://community.rockrms.com/developer/rock-api) · [Rock developer hub](https://community.rockrms.com/developer) · [Canonical spec hub](https://saltlight-richard.github.io/saltlight-specs/specs/rock-rms.html) · live instance [saltlight.rockcloud.com](https://saltlight.rockcloud.com) (Rock 18.1.0.0). Swagger is browsable per controller — open `https://saltlight.rockcloud.com/api/docs` and pick the named controller (WorkflowTypes, Attributes, AttributeQualifiers, DefinedValues, Tags, Campuses).

**Stored config shape** — `workflow.workflow_data['rockrms']` (a JSONB array; one element in v1). No field mapping is stored here; the mapping lives at the account level (Feature 1b).

```json
[
  {
    "workflow_type_id": 24,
    "trigger_attribute_id": 8286,
    "trigger_match_mode": "equals",
    "trigger_values": ["Sunday Service"],
    "writeback_tag_ids": [1],
    "writeback_workflow_type_ids": [24]
  }
]
```

- `workflow_type_id` — the mapped form chosen in Section A (A1) (the WorkflowType id; 24 = "First Time Guest - SaltLight" on the test instance).
- `trigger_attribute_id` / `trigger_match_mode` / `trigger_values` — Section A (A2); **required for any Rock-connected CareFlow** — together they identify the selection that fires this CareFlow. `trigger_match_mode` is `"equals"` (single-select field) or `"any_of"` (multi-select/checkbox field), auto-detected from the attribute's `FieldTypeId` and admin-confirmable; Feature 3 reads it to choose exact-match vs intersection-match. `trigger_values` is always an array of one or more strings (store the Campus Id-string for a campus field, e.g. `"1"`). InterestArea (AttributeId 8286) is Feature 2's trigger field — it is NOT one of the five mapped core fields.
- `writeback_workflow_type_ids` / `writeback_tag_ids` — Section B (B1 workflows, B2 tags); multi-select arrays; both optional. Empty/absent means no write-back. These are consumed by Feature 4. (WorkflowType 24, Tag Id 1 = "Staff" on the test instance.)

> **Key-name note for Kenneth:** Feature 2 uses the **plural, array** keys `writeback_tag_ids` and `writeback_workflow_type_ids` because write-back is multi-select (confirmed in the corrected guidance). These supersede any singular `writeback_tag_id` / `writeback_workflow_type_id` shown in earlier draft schema sections. Feature 4 must read the plural array keys.

**Example dropdown response shape** (what each lookup handler returns to the portal — a bare list, never a `{"value":[...]}` wrapper):

```json
[
  { "id": "24", "name": "First Time Guest - SaltLight" }
]
```

### Gotchas
- **No field mapping here.** Step 6 must not ask which Rock field maps to first name/email/etc. That is Feature 1b (account level, stored in `account_metadata['rockrms']['mapped_forms']`). Feature 2 only reads the already-mapped forms list and stores per-CareFlow choices.
- **`workflow_data['rockrms']` is an ARRAY**, not an object. v1 stores a single element, but read and write it as an array so the shape never has to change later.
- **Write-back is multi-select.** Tags and workflow types are arrays (`writeback_tag_ids`, `writeback_workflow_type_ids`), not single ids. Store arrays even when the admin picks one.
- **Every lookup endpoint used here returns a BARE ARRAY**, not `{"value":[...]}` — WorkflowTypes, Attributes, AttributeQualifiers, DefinedTypes, DefinedValues, Tags, and Campuses all come back as a plain list. Treat them all the same way. (On this install every Rock list endpoint returns a bare array; never assume a `value` wrapper.)
- **EntityTypeId 113 is the Workflow entity type — verified.** The form-fields lookup filters `EntityTypeId eq 113` because workflow attributes belong to the Workflow entity type (Id 113, confirmed in the Test Environment doc). This is a verified value, not a magic number — keep it.
- **Tags are Person tags only.** The Tags lookup must filter `EntityTypeId eq 15` (Person). Without it the list includes tags for other entity types and the write-back will target the wrong entity. (Feature 4's `POST /api/TaggedItems` likewise requires `EntityTypeId: 15`.)
- **CampusPicker trigger values are stored as the Campus Id string.** If an admin chooses a campus field as the trigger attribute, Rock stores that attribute's value as the Campus **Id as a string** (e.g. `"1"`), not the campus name. The trigger-values multi-select should display campus names but persist the Id string so Feature 3's comparison matches. Resolve names via the campus cache at `account_metadata['rockrms']['campuses']` (written at connect time).
- **GUID filter syntax** when resolving a DefinedValue picker's option list: the DefinedType lookup must use `guid'XXXXXXXX-...'`, not a bare quoted string, or it returns no rows.
- **Value List fields** store their options as a single pipe-delimited string of `Label,value` pairs in the `values` qualifier (e.g. `Sunday Service,Sunday Service|Youth Ministry,Youth Ministry`). Split on `|`, then on the first comma, to build the trigger-values options.
- **Saving Step 6 performs no Rock write.** The only Rock traffic at config time is read-only dropdown population. Applying tags and launching workflows happens later in Feature 4 — do not fire any TaggedItems or Workflows POST from this screen.
- **Trigger-value comparison is exact-string.** Feature 3 compares the submission's attribute value to the stored `trigger_values` as strings, so store exactly what Rock returns for that attribute (including the Campus Id-string case above). Note Rock returns and stores timestamps in **local server time, not UTC**, so any future timestamp-based filtering Feature 3 does (Track 2 advances on `CreatedDateTime`) is its concern, not Step 6's.
## Feature 3 — Guest Intake: Rock → SaltLight (ingestion only)

> 🧭 Rock RMS Integration › Feature 3 — Guest Intake: Rock → SaltLight (ingestion only)

### User story

As a church admin, I want new visitors and completed guest forms from Rock to automatically appear in SaltLight so that my care team can begin follow-up without anyone re-entering the data.

### User journey

This feature is a single scheduled Lambda that polls Rock on a schedule and ingests guests through two independent tracks. It **never writes anything back to Rock** — all write-back (creating/updating people, applying tags, creating workflows) lives in Feature 4. Rock-sourced Connect Cards are tagged by source so Feature 4 can recognise them and skip them, preventing a sync loop.

**Ingestion is polling-only in v1.** The two tracks below (scheduled poll of People + Workflows) are the *only* ingestion paths that ship. There is **no Rock webhook receiver in v1** — a future (v2) webhook path would create cards tagged `source: "rock_webhook"`, which is why Feature 4's source guard already knows that value, but no `rock_webhook` card is produced anywhere in v1. The `org_id` stored at connect time (Feature 1) is reserved for routing that future webhook path; **Feature 3 does not read `org_id` and has no webhook fallback**.

**The poll gate (runs first, every time)**

1. An EventBridge rule fires the Lambda **every 5 minutes, always** — for every connected account, on every day.
2. For each connected account, the Lambda reads that account's `poll_schedule` and runs a **should-poll check** (evaluated in the account's configured `timezone`):
   - If today is a high-frequency day (default Saturday + Sunday, Python weekdays `[5, 6]`) **and** the current time is inside the service window (`high_frequency_start`–`high_frequency_end`), poll now (this is the "every 5 minutes on service days" behaviour).
   - Otherwise, poll only if the low-frequency interval (default `low_frequency_interval_hours: 12` = twice daily) has elapsed since `last_polled`.
   - If neither condition is met, skip this account this cycle — do nothing, log a skip, move on.
3. The schedule is **per-account configurable**: a church with Wednesday-night services can add Wednesday to its high-frequency days.
4. After a poll actually runs, `last_polled` is updated to the run time.

**Track 1 — new/updated Rock Person records**

1. A volunteer or staff member adds (or edits) a Person directly in Rock — at a kiosk or via data entry.
2. The poller fetches People whose `ModifiedDateTime` is greater than the Track 1 cursor, expanding `PhoneNumbers`, ordered by `ModifiedDateTime` ascending. (No `$select` — see Gotchas; `$select` strips the expanded phone numbers.)
3. For each person returned, the poller **dedups by Rock person id** — if a Connect Card already exists for that `rock_person_id` on this account, skip the person.
4. For a new person, the poller reads the mobile phone from the expanded `PhoneNumbers` (the entry whose `NumberTypeValueId` matches the account's cached `mobile_id`), resolves the campus name from the account's campus cache using `PrimaryCampusId`, and creates a Connect Card tagged `source: "rock_track1"`. `PrimaryCampusId` is often null on this install, so campus may be absent — read it defensively.
5. The Connect Card is matched to the CareFlow(s) configured in Step 6 (Feature 2) and the CareFlow begins — the assigned leader is notified.
6. After all people are processed, the Track 1 cursor advances to the **maximum `ModifiedDateTime`** seen this run.

**Track 2 — completed Rock workflow form submissions**

1. A guest completes the church's existing connect-card form in Rock — a Workflow of a mapped WorkflowType (e.g. "First Time Guest - SaltLight", type id 24).
2. For each mapped WorkflowType, the poller fetches **completed** Workflow instances of that type whose `CreatedDateTime` is greater than the Track 2 cursor, ordered by `CreatedDateTime` ascending. `Status eq 'Completed'` goes **in the `$filter`** alongside `WorkflowTypeId` and `CreatedDateTime` — this is verified working on Rock 18.1.0.0 (confirmed 2026-06-02 returning workflows 8/9/10), so there is no in-memory status pass. (Each WorkflowType is polled once per cycle even if several CareFlows reference it.)
3. Because Rock will not return a workflow's field values inline (`$expand=AttributeValues` is rejected with HTTP 400), the poller then makes a **batch fetch** of `/api/AttributeValues` for all relevant attribute ids, and **joins the values to their workflows in memory** by `EntityId` (= the workflow id).
4. For each workflow, the poller **dedups by Rock workflow id** — if a Connect Card already exists for that `rock_workflow_id`, skip it.
5. Using the Feature 1b field mapping (`account_metadata['rockrms']['mapped_forms']`), the poller reads first name, last name, email, phone and campus off the joined attribute values, plus the CareFlow's trigger field (Feature 2). A CampusPicker value arrives as the Campus **Id as a string** (e.g. `"1"`) and must be resolved to a name through the campus cache. The poller then applies the CareFlow's **required trigger selection** using its **`trigger_match_mode`**: for `"equals"` (single-select fields) the submission's single selection must equal one of the CareFlow's `trigger_values`; for `"any_of"` (multi-select/checkbox fields, where Rock stores several selections in one delimited attribute value) split the stored value and keep the submission if **any** checked value is in the CareFlow's `trigger_values`. If nothing matches, skip the submission for this CareFlow (the guest did not make the selection that fires it).
6. A surviving workflow becomes a Connect Card tagged `source: "rock_track2"`, matched to the CareFlow, and the CareFlow begins.
7. After all mapped types are processed, the Track 2 cursor advances to the **maximum `CreatedDateTime`** seen this run (accumulated across all WorkflowTypes, written once at the end).

**On error**

- If Rock is unreachable during a poll, the Lambda logs the error for that account and moves on to the next account — one account's failure never aborts the others, and the run never crashes. The cursor for a failed account is not advanced, so the next successful poll picks up everything since the last good cursor.

### Acceptance criteria

- [ ] A new or modified Rock Person appears as a `rock_track1` Connect Card within 5 minutes during a high-frequency service window.
- [ ] A **Completed** Rock Workflow of a mapped type appears as a `rock_track2` Connect Card within 5 minutes during a high-frequency service window; a non-Completed (e.g. Active) workflow of the same type does **not** produce a Connect Card (it is excluded by the `Status eq 'Completed'` filter, so it never comes back from Rock).
- [ ] The same Rock person never produces two Connect Cards (Track 1 dedup by `rock_person_id`); the same Rock workflow never produces two Connect Cards (Track 2 dedup by `rock_workflow_id`).
- [ ] On a non-high-frequency day/time, an account is polled at most twice per day (low-frequency interval), and skipped cycles are observable in logs.
- [ ] Track 1 filters and sorts by `ModifiedDateTime`, and its cursor advances to the maximum `ModifiedDateTime` seen — not `CreatedDateTime`.
- [ ] Track 2 filters and sorts by `CreatedDateTime`, includes `Status eq 'Completed'` in the `$filter`, and its cursor advances to the maximum `CreatedDateTime` seen.
- [ ] Track 2 reads each submission's fields via a batched `/api/AttributeValues` call joined in memory by `EntityId`; the run never uses `$expand=AttributeValues`.
- [ ] A Track 2 CampusPicker value (Campus Id as a string, e.g. `"1"`) is resolved to a campus name via the cached campuses before being stored on the Connect Card.
- [ ] When a CareFlow defines a trigger field + allowed values, only matching submissions become Connect Cards — **exact match** for a single-select trigger (`trigger_match_mode: "equals"`) and an **intersection match** for a multi-select trigger (`trigger_match_mode: "any_of"`, where the guest's checked values are split and any overlap with the allowed set qualifies). A guest who checks several boxes including an allowed one still qualifies.
- [ ] If Rock is unreachable for one account, that account is logged and skipped, other accounts still poll, the Lambda does not crash, and the failed account's cursor is not advanced.
- [ ] Every Connect Card created here carries a Rock source tag (`rock_track1` / `rock_track2`), and nothing in this feature writes to Rock. (No `rock_webhook` card is created in v1 — that source is reserved for a future webhook path.)

### What the backend touches

- **Lambda / handler:** the scheduled poll handler in `tf/lambda_functions/orchestrator/src/rockrms.py` (the always-5-min EventBridge target), which iterates connected accounts and, for each that passes the should-poll gate, runs the per-account Track 1 + Track 2 ingestion. Registered in the orchestrator's scheduled-event function registry in `main.py`; the every-5-minute rule lives in `tf/events.tf`.
- **Rock client methods:** add poll/fetch helpers to `tf/lambda_functions/shared/integrations/rockrms.py` — fetch new/updated People (Track 1), fetch completed Workflows of a type (Track 2 call 1), batch-fetch AttributeValues (Track 2 call 2), plus the shared bare-array normaliser used on every list response.
- **`public.account` → `account_metadata['rockrms']` JSONB:** reads `connected`, `campuses`, `mobile_id`, `poll_schedule`, `last_polled`; reads/writes `poll_cursor_track1` and `poll_cursor_track2`. (The `campuses` cache is populated at connect time by Feature 1 — Feature 3 only reads it.)
- **`public.workflow` → `workflow_data['rockrms']` JSONB:** reads each CareFlow's Rock config — which mapped form (WorkflowTypeId) and the trigger field + allowed values (Feature 2).
- **`public.account_metadata['rockrms']['mapped_forms']` (Feature 1b):** the account-level form → field-id mapping used to read Track 2 submission values (first name, last name, email, phone, campus).
- **`public.connect_card` → `connect_card_data` JSONB:** inserts new Connect Cards tagged by `source`; also the dedup lookups (`connect_card_data->>'rock_person_id'` for Track 1, `connect_card_data->>'rock_workflow_id'` for Track 2).
- **`public.interaction`:** one pending interaction row per configured CareFlow so the CareFlow engine starts follow-up.
- **No write-back endpoints are called by this feature.** Ingestion only. All POSTs/PATCHes to Rock (create/update person, `POST /api/TaggedItems`, create workflows) live in Feature 4.

### Reference material

| What | Endpoint (+ key query params) | Method | Direct links |
|---|---|---|---|
| Fetch new/updated people (Track 1) | `/api/People?$filter=ModifiedDateTime gt datetime'<cursor>'&$expand=PhoneNumbers&$orderby=ModifiedDateTime asc&$top=500` (no `$select`) | GET | [Swagger › People](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Track 1: People poll"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| Fetch completed workflow submissions of a type (Track 2, call 1) | `/api/Workflows?$filter=WorkflowTypeId eq <typeId> and Status eq 'Completed' and CreatedDateTime gt datetime'<cursor>'&$orderby=CreatedDateTime asc&$top=500` | GET | [Swagger › Workflows](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Track 2 call 1: Workflows poll"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| Batch-fetch submission field values (Track 2, call 2) | `/api/AttributeValues?$filter=(AttributeId eq 8281 or … or AttributeId eq 8286)&$select=AttributeId,EntityId,Value&$top=5000` | GET | [Swagger › AttributeValues](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Track 2 call 2: AttributeValues batch"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| Campus cache (resolve campus id → name) | `/api/Campuses?$filter=IsActive eq true&$select=Id,Guid,Name,ShortCode` (read at connect time in Feature 1; cached in `account_metadata`; Feature 3 reads the cache) | GET | [Swagger › Campuses](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Campuses list"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| OData conventions ($filter, $expand, $select, $orderby, $top) | n/a | n/a | [Rock REST API overview](https://community.rockrms.com/developer/rock-api) · [Rock developer hub](https://community.rockrms.com/developer) |
| Canonical spec hub | n/a | n/a | [Rock RMS spec hub](https://saltlight-richard.github.io/saltlight-specs/specs/rock-rms.html) |

Test data ready to pull (created 2026-06-02 on `saltlight.rockcloud.com`, Rock 18.1.0.0): Track 1 People **22 (PollerTest Alpha), 23 (Bravo), 24 (Charlie)**, each with a mobile phone; Track 2 Workflows **8 (Delta / Sunday Service), 9 (Echo / Youth Ministry), 10 (Foxtrot / Kids Ministry)**, all `Status=Completed` with all 6 attributes populated. Suggested cursor `2026-06-02T12:11:00` (Rock-local, no trailing `Z`) catches all six. WorkflowType 24 attribute ids: FirstName 8281, LastName 8282, Email 8283, Phone 8284, Campus 8285, InterestArea 8286 (InterestArea is Feature 2's trigger field, not one of the five mapped core fields). Campuses: 1 = Main, 2 = North. Ignore stale People 20–21 (Rock blocks DELETE on `/api/People`).

**Track 1 — People response shape** (note: a **bare array**, and `PhoneNumbers` arrives inline only because there is no `$select`; timestamps are Rock-LOCAL time, no trailing `Z`):

```json
[
  {
    "Id": 22,
    "FirstName": "PollerTest",
    "LastName": "Alpha",
    "Email": "alpha@example.com",
    "ModifiedDateTime": "2026-06-02T12:11:00",
    "PrimaryCampusId": 1,
    "PhoneNumbers": [
      { "Number": "(555) 201-1001", "NumberTypeValueId": 12 }
    ]
  }
]
```

**Track 2 — AttributeValues response shape** (a **bare array**; join to the workflow by `EntityId`; a CampusPicker value arrives as the Campus **Id as a string**):

```json
[
  { "AttributeId": 8281, "EntityId": 8, "Value": "Delta" },
  { "AttributeId": 8285, "EntityId": 8, "Value": "1" },
  { "AttributeId": 8286, "EntityId": 8, "Value": "Sunday Service" }
]
```

**Connect Card stored shapes** (data, not logic):

```json
{
  "source": "rock_track1",
  "rock_person_id": 22,
  "rock_data": {
    "first_name": "PollerTest",
    "last_name": "Alpha",
    "email": "alpha@example.com",
    "phone": "(555) 201-1001",
    "campus_id": 1,
    "campus_name": "Main"
  }
}
```

```json
{
  "source": "rock_track2",
  "rock_workflow_id": 8,
  "rock_workflow_type_id": 24,
  "rock_data": {
    "first_name": "Delta",
    "last_name": "Guest",
    "email": "delta@example.com",
    "phone": "(555) 201-1001",
    "campus_name": "Main"
  }
}
```

**Poll schedule + cursor config** (in `account_metadata['rockrms']`, read by the should-poll gate):

```json
{
  "poll_schedule": {
    "high_frequency_days": [5, 6],
    "high_frequency_start": "15:00",
    "high_frequency_end": "21:00",
    "low_frequency_interval_hours": 12,
    "timezone": "America/Chicago"
  },
  "last_polled": "2026-06-02T12:11:00",
  "poll_cursor_track1": "2026-06-02T12:11:00",
  "poll_cursor_track2": "2026-06-02T12:11:00"
}
```

`high_frequency_days` are Python weekday values: 5 = Saturday, 6 = Sunday. The window (`high_frequency_start`/`high_frequency_end`) and `low_frequency_interval_hours` are evaluated in the configured `timezone`. EventBridge fires every 5 minutes for every account; the should-poll gate decides whether this cycle actually polls.

### Gotchas

- **Track 1 uses `ModifiedDateTime`, not `CreatedDateTime`** — for both the `$filter` and the `$orderby`, and the cursor advances to the max `ModifiedDateTime` seen. This is so edits to an existing Rock person (not just brand-new records) are picked up. (Track 2 uses `CreatedDateTime` instead, because a completed workflow's creation time *is* its submission time.)
- **Track 2 puts `Status eq 'Completed'` in the `$filter` — do NOT filter status in memory.** The full Track 2 query is `WorkflowTypeId eq <typeId> and Status eq 'Completed' and CreatedDateTime gt datetime'<cursor>'`, ordered by `CreatedDateTime asc`. This is verified working on Rock 18.1.0.0 (confirmed 2026-06-02 returning workflows 8/9/10). Filter, order, and cursor advance all use `CreatedDateTime`; never substitute `ModifiedDateTime` for Track 2.
- **Rock returns LOCAL server time, not UTC.** A record created at 19:11 UTC comes back stamped 12:11 on this install (~7h behind). Store every cursor as a plain `YYYY-MM-DDThh:mm:ss` string on Rock's clock — **no trailing `Z`, no offset** — so it drops straight into the `datetime'...'` filter. If you build the filter from a UTC clock, it lands in the future and silently returns an empty list.
- **All Rock list endpoints return a bare array `[...]`, not `{"value":[...]}`.** This applies to `/api/People`, `/api/Workflows`, and `/api/AttributeValues` too — do not read a `value` key off the response. (Any older test-env note claiming People/Workflows are `{"value":[...]}` is wrong; use the bare-array normaliser everywhere.)
- **OData datetime filters must be `datetime'YYYY-MM-DDThh:mm:ss'`** — a bare ISO string returns HTTP 400.
- **Never combine `$select` with `$expand=PhoneNumbers`** — `$select` silently strips the expanded `PhoneNumbers`, leaving Track 1 with no phone numbers. Omit `$select` on the People poll entirely.
- **Never use `$expand=AttributeValues` on `/api/Workflows`** — it returns HTTP 400 always. Track 2 must do the separate batched `/api/AttributeValues` call and join by `EntityId` in memory.
- **Batch the AttributeValues call by AttributeId, not by EntityId.** Filtering by EntityId (workflow id) makes the URL grow without bound and exceed server limits at scale; fetch by the small fixed set of attribute ids (`(AttributeId eq 8281 or … or AttributeId eq 8286)`) and join to workflows client-side.
- **CampusPicker stores the Campus Id as a string** (e.g. `"1"`), not the campus name — resolve it through the campus cache to get a name.
- **A multi-select (checkbox) trigger field stores several selections in one delimited AttributeValue** — when the CareFlow's `trigger_match_mode` is `"any_of"`, split the stored value and test for set intersection with the allowed values, never an exact string compare, or a guest who checked extra boxes is silently skipped. The exact delimiter Rock writes for a multi-select value is **Kenneth-verify** (the loaded test data is single-valued); confirm it against a live multi-select submission before relying on the split character.
- **`PrimaryCampusId` is often null on this install** and `$expand=Campus` on People is silently ignored (campus lives in the Family Group model). Read campus defensively; a Track 1 card may have no campus.
- **Single-record GETs ignore `$select`** (they return the full object) — read individual fields defensively rather than assuming a trimmed shape.
- **`$top=500` is the per-call ceiling** for People/Workflows (`$top=5000` for the AttributeValues batch). If exactly 500 records come back, records sharing the boundary second may be missed this cycle; the next cycle's cursor catches them. Acceptable at current scale.
- **Dedup must run before insert, per track** — Track 1 by `rock_person_id`, Track 2 by `rock_workflow_id`. A guest who submits the same Rock form twice, or a poll that overlaps a cursor boundary, must still produce only one Connect Card.
- **Source tagging is the loop guard.** Every card created here is tagged `rock_track1` / `rock_track2`; Feature 4 keys off that source to skip pushing these cards back to Rock. Do not omit or rename the source. (`rock_webhook` is reserved for a possible future v2 webhook path and is never produced in v1 — the guards are forward-compatible only.)
- **One failing account must not abort the run.** Wrap each account's poll so an unreachable Rock or a bad cursor logs and continues; never advance a cursor for an account whose poll did not complete.
## Feature 4 — Card Capture Push: SaltLight → Rock

> 🧭 Rock RMS Integration › Feature 4 — Card Capture Push: SaltLight → Rock

**Scope:** write-back ONLY. This feature owns every write into Rock (search, create/update person, apply tags, create workflows). It does not poll Rock or create Connect Cards from Rock — that is Feature 3 (ingestion only). Feature 4 fires when a Connect Card is captured in SaltLight from a non-Rock source.

### User story

As a church admin, I want guests captured by SaltLight (scanned or manually entered) to automatically appear in my Rock database — matched to the right person and attributed with the configured tags and follow-up workflows — so that my system of record stays current without anyone entering data twice, and so a Rock outage never holds up the guest's follow-up.

### User journey

A Connect Card is created in SaltLight from a non-Rock source (CardCapture scan, manual entry, or web form). The CareFlow it matched has Rock write-back configured in its Feature 2 **Section B (outbound)** settings. SaltLight attempts to push the guest to Rock immediately, but the CareFlow starts regardless — the push never blocks follow-up.

**Track A — Rock is available (the happy path)**

1. SaltLight checks the card's `source`. If it is `rock_track1` or `rock_track2` (the two v1 ingestion sources), it stops here — that card came FROM Rock and must never be pushed back (that would create a loop). The guard also recognizes `rock_webhook`, a reserved-for-future source with no v1 producer (see Gotchas) — so the guard is forward-compatible the day a webhook ingestion path ships, but no `rock_webhook` card exists in v1.
2. SaltLight reads the CareFlow's write-back config (`writeback_tag_ids`, `writeback_workflow_type_ids`). If both are empty, there is no write-back configured — nothing to do, and it stops.
3. SaltLight marks the card `rock_sync_status = 'pending'` before doing anything else, so any failure leaves it queued.
4. **Find the person.** SaltLight searches Rock by **email first**. If no email match (or no email on the card), it searches by **phone, digits-only**. The first match wins.
5. **Found → update.** SaltLight updates the existing Rock person with any new non-blank fields from the card. It never overwrites an existing Rock value with a blank. (v1 updates first name / last name / email only — see Gotchas for why phone is excluded.)
6. **Not found → create.** SaltLight creates a new Rock Person using the card fields, stamped as a Visitor (ConnectionStatus from the connect-time cache) with an Active record status. `POST /api/People` returns the new person's Id as a **bare integer** — read it as the return value directly (see Gotchas).
7. SaltLight stores the resulting Rock person Id on the SaltLight person record (`person_metadata['rockrms_person_id']`) so future pushes reuse the same person.
8. **Apply tags.** For each id in `writeback_tag_ids`, SaltLight resolves the person's Guid (via a person GET), then applies that Tag to the person in Rock.
9. **Create workflows.** For each id in `writeback_workflow_type_ids`, SaltLight creates one Workflow instance in Rock for that person (associated via the person's `PrimaryAliasId`, not the raw person Id) and fills its attributes (name, email, phone, campus, interest) from the card.
10. SaltLight marks the card `rock_sync_status = 'synced'`. The guest is now in SaltLight's CareFlow AND attributed in Rock.

**Track B — Rock is unavailable (queue + retry)**

1. Steps 1–3 above still run. The card is already marked `'pending'`.
2. The push attempt throws (Rock unreachable / errored). SaltLight increments `rock_sync_attempts`, records `rock_sync_last_attempted`, and leaves the card `'pending'`. The CareFlow has already started — the guest is unaffected.
3. On every later poll cycle (Feature 3's scheduled Lambda), SaltLight sweeps that account's `'pending'` cards and retries each one through the exact same find/create/tag/workflow sequence.
4. When a retry succeeds, the card flips to `'synced'`.
5. If a card crosses **10 consecutive failed attempts**, SaltLight flips it to `'failed'` and alerts SaltLight's internal team for manual investigation. The CareFlow is still untouched.

**Examples (verifiable against the loaded test data):**

- Maria scanned at the welcome tent visited 6 months ago and is already in Rock by email. Her record is updated with today's contact info (no blanks written), the configured tags are applied, and a "First Time Guest - SaltLight" workflow (Type 24) is created with her name, email, phone, campus, and her circled interest filled into attributes 8281–8286.
- David is brand new. No email or phone match, so SaltLight creates him in Rock as a Visitor (the create returns his new Id as a bare integer), then applies tags and creates the workflows.
- Rock is down when Emily's card is scanned. Her Connect Card and CareFlow start immediately; her push stays `'pending'` and is retried on the next poll. If it can't land in 10 tries, the team is alerted.

### Acceptance criteria

- [ ] A non-Rock-sourced card whose CareFlow has write-back configured results, when Rock is available, in the guest existing in Rock within seconds of Connect Card creation.
- [ ] Search order is enforced: email first; phone (digits-only) only if email did not match or was absent.
- [ ] When the person already exists in Rock (matched by email or phone), no duplicate Person is created.
- [ ] On update, no existing Rock field is overwritten with a blank — only non-empty card fields are written.
- [ ] For each id in `writeback_tag_ids`, the corresponding Tag is applied to the person; for each id in `writeback_workflow_type_ids`, one Workflow instance is created and its name/email/phone/campus/interest attributes are populated from the card.
- [ ] On create, the new person's Id is read from the bare-integer return of `POST /api/People` and saved to `person_metadata['rockrms_person_id']` so subsequent pushes for that guest reuse the same Rock person.
- [ ] When Rock is unavailable, the card is marked `'pending'`, the CareFlow still starts, and the card is retried on later poll cycles until it succeeds.
- [ ] After 10 consecutive failed attempts for the same card, status becomes `'failed'` and SaltLight's internal team is alerted.
- [ ] A card whose `source` is `rock_track1` or `rock_track2` (or the reserved-for-future `rock_webhook`) is never pushed to Rock (status stays untouched; no Rock write occurs).

### What the backend touches

- **Trigger point:** the Connect Card creation path (`connect_card_tools.py`) calls an "attempt Rock sync if configured" step right after the card row is written. This step sets `rock_sync_status = 'pending'` first, then attempts the push; it is fire-and-record, never blocking the CareFlow.
- **Core write-back:** the outbound sync lives on the `RockRMS` integration class in `tf/lambda_functions/shared/integrations/rockrms.py` — search-by-email, search-by-phone, update-person, create-person, apply-tag, and create-workflow-with-attributes all belong here. This also replaces the old stubbed `associate_person_to_rockrms_workflow()` (Feature 4 owns it).
- **Retry sweep:** lives in the orchestrator poll handler `tf/lambda_functions/orchestrator/src/rockrms.py`, called at the end of each account's poll run (the Feature 3 schedule, every 5 min gated by `_should_poll_now`). It selects that account's `'pending'` cards and re-runs the same push.
- **Source guard:** applied at the start of both the immediate trigger and the sweep — `source ∈ {rock_track1, rock_track2, rock_webhook}` short-circuits with no Rock write. Only `rock_track1` and `rock_track2` exist in v1; `rock_webhook` is listed so the guard is forward-compatible, but no v1 path produces it.
- **Tables / JSONB keys:**
  - `public.connect_card` — read `connect_card_data` (source + card fields); write the three sync-tracking columns `rock_sync_status`, `rock_sync_attempts`, `rock_sync_last_attempted` (new columns; require a migration before deploy).
  - `public.workflow.workflow_data['rockrms']` — read the CareFlow's write-back config: `writeback_tag_ids` (array) and `writeback_workflow_type_ids` (array), plus the attribute-id mapping used to fill workflow fields.
  - `public.account.account_metadata['rockrms']` — read `connected`, `campuses` (campus cache, for resolving a CampusPicker value to a Campus Id), and `visitor_id` (ConnectionStatus for new people).
  - `public.person.person_metadata['rockrms_person_id']` — write the matched/created Rock person Id.
- **Endpoints touched in Rock:** People (search/get/create/update), TaggedItems (apply tag), Workflows (create instance), AttributeValues (fill workflow attributes). All listed below.

### Reference material

| What | Endpoint (+ key query params) | Method | Direct links |
|---|---|---|---|
| Search person by email | `/api/People?$filter=Email eq '<email>'&$top=5&$select=Id,FirstName,LastName,PrimaryAliasId` | GET | [Swagger › People](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Write-back: search by email"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| Search person by phone (digits only) | `/api/People?$filter=PhoneNumbers/any(p: p/Number eq '<digits>')&$top=5&$select=Id,FirstName,LastName,PrimaryAliasId` | GET | [Swagger › People](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Write-back: search by phone (verify OData any)"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| Get person (resolve Guid + PrimaryAliasId) — single-record GET ignores `$select`, returns the full object; read fields defensively | `/api/People/<id>?$select=Id,Guid,PrimaryAliasId` | GET | [Swagger › People](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Write-back: get alias"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| Update existing person (non-blank fields only) | `/api/People/<id>` | PATCH | [Swagger › People](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Write-back: update person"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| Create new person (returns new Id as a bare integer) | `/api/People` | POST | [Swagger › People](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Write-back: create person"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| Apply a tag to the person | `/api/TaggedItems` (body requires `EntityTypeId: 15`; `EntityGuid` is the person's Guid) | POST | [Swagger › TaggedItems](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Write-back: apply tag"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| Create a workflow instance | `/api/Workflows` (body sets `InitiatorPersonAliasId` to the person's `PrimaryAliasId`; returns new Id as a bare integer) | POST | [Swagger › Workflows](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Write-back: create workflow"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| Check if a workflow attribute value row exists | `/api/AttributeValues?$filter=EntityId eq <workflow_id> and AttributeId eq <attr_id>&$select=Id` | GET | [Swagger › AttributeValues](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Write-back: AV check"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| Create a workflow attribute value (returns new Id as a bare integer) | `/api/AttributeValues` | POST | [Swagger › AttributeValues](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Write-back: AV create"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| Update an existing workflow attribute value | `/api/AttributeValues/<id>` | PATCH | [Swagger › AttributeValues](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Write-back: AV update"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |

General references: [Rock REST API overview](https://community.rockrms.com/developer/rock-api) · [Rock developer hub](https://community.rockrms.com/developer) · [Rock O/Swagger explorer (live on the install)](https://saltlight.rockcloud.com/api/docs) · [Rock RMS Test Environment doc (credentials, verified curls, IDs, GUIDs)](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) · [Canonical spec hub](https://saltlight-richard.github.io/saltlight-specs/specs/rock-rms.html)

**CareFlow write-back config shape** (read from `workflow_data['rockrms']`; both arrays optional — empty means skip that kind of write-back):

```json
{
  "writeback_tag_ids": [1, 42],
  "writeback_workflow_type_ids": [24],
  "attribute_ids": {
    "first_name": 8281,
    "last_name": 8282,
    "email": 8283,
    "phone": 8284,
    "campus": 8285,
    "interest_area": 8286
  }
}
```

**Connect-card sync-tracking columns** (data, not logic):

```json
{
  "rock_sync_status": "pending | synced | failed | null",
  "rock_sync_attempts": 0,
  "rock_sync_last_attempted": "2026-06-02T12:11:00"
}
```

`null` status = write-back not applicable (no Rock connection, or the card came from Rock). `'failed'` = crossed the 10-attempt threshold; team alerted. The `rock_sync_last_attempted` value, like every timestamp Rock hands back, is Rock-local server time, not UTC — store it as a plain `YYYY-MM-DDThh:mm:ss` string with no trailing `Z` and no offset (see Gotchas).

**Apply-tag request shape** (note required `EntityTypeId`; `EntityGuid` is the person's Guid, not their Id):

```json
{ "TagId": 42, "EntityGuid": "<person Guid>", "EntityTypeId": 15 }
```

**Create-person request shape** (stamps a new Visitor with Active status; `ConnectionStatusValueId` comes from the connect-time cache `account_metadata['rockrms']['visitor_id']`):

```json
{
  "FirstName": "David",
  "LastName": "Nguyen",
  "Email": "david@example.com",
  "ConnectionStatusValueId": 66,
  "RecordTypeValueId": 1,
  "RecordStatusValueId": 3
}
```

The response to this POST is a **bare integer** (the new person Id), e.g. `25` — not a JSON object.

**Test data to verify against** (loaded 2026-06-02): WorkflowType 24 = "First Time Guest - SaltLight"; attributes FirstName 8281, LastName 8282, Email 8283, Phone 8284, Campus 8285, InterestArea 8286; Campuses 1 = Main, 2 = North; Tag Id 1 = "Staff".

### Gotchas

- **Never push Rock-sourced cards back.** Cards with `source` of `rock_track1` or `rock_track2` came from Rock's two v1 ingestion tracks. Pushing them back creates an infinite loop. Guard at every entry point (immediate trigger and retry sweep).
- **`rock_webhook` is reserved-for-future, not a v1 source.** Webhooks are OUT OF SCOPE for v1 — there is no webhook receiver and no path that produces a `rock_webhook`-sourced card. The source guard still lists `rock_webhook` so it stays forward-compatible the day a v2 webhook ingestion path ships, but in v1 the only Rock-sourced cards are `rock_track1` and `rock_track2`. Do not build a webhook receiver here, and do not treat `org_id` as a load-bearing webhook routing key — it is stored at connect time for possible future routing only.
- **All Rock list endpoints return BARE ARRAYS**, not `{"value":[...]}`. This is true across the board — `/api/People`, `/api/Workflows`, `/api/AttributeValues`, `/api/Tags`, every list endpoint. Treat every search/lookup result as a plain array. (The old test-env doc claiming People/Workflows wrap in `{"value":...}` is wrong — verified on Rock 18.1.0.0.)
- **POSTs return a BARE INTEGER**, not an object. `POST /api/People`, `POST /api/Workflows`, and `POST /api/AttributeValues` each return just the new row's Id (e.g. `25`). Do not call `.get('Id')` on the result — read the return value directly; the write succeeded if no exception was raised. (Reading the Id defensively is fine as a safety net, but the primary path is a bare int on this install.)
- **PATCH returns HTTP 204 with no body.** `PATCH /api/People/<id>` and `PATCH /api/AttributeValues/<id>` succeed with an empty 204 — do not try to parse a response body.
- **Single-record GETs ignore `$select`.** `GET /api/People/<id>?$select=...` returns the full person object regardless of the `$select` list. The fields you asked for are present, plus everything else — read defensively, don't assume a trimmed object.
- **TaggedItems requires `EntityTypeId: 15` (Person).** Omitting it returns HTTP 500. Rock also does NOT dedupe TaggedItems — duplicate POSTs create duplicate rows (harmless; Rock dedupes on display), so no pre-check is needed.
- **TaggedItems uses the person's Guid, not their Id.** Resolve the Guid via a GET on the person first; the apply-tag body needs `EntityGuid`.
- **Workflows associate via PersonAlias, not PersonId.** When creating a workflow instance for a person, use their `PrimaryAliasId` (from the person GET), not the raw person Id.
- **Workflow attribute rows are created lazily.** A brand-new workflow instance has NO AttributeValue rows yet. To fill an attribute: check whether a row exists for that `(EntityId, AttributeId)` — if yes, PATCH it; if no, POST a new one. The convenience endpoints `PUT /api/Workflows/{id}/SetAttributeValue` and `POST /api/Workflows/{id}/SaveAttributeValues` are BROKEN (404 on this install) — do not use them. (Workflow attribute definitions themselves are read via `GET /api/Attributes?$filter=EntityTypeId eq 113 and EntityTypeQualifierColumn eq 'WorkflowTypeId' and EntityTypeQualifierValue eq '<typeId>'` — EntityTypeId 113 is the verified Workflow entity type, not a magic number.)
- **CampusPicker stores the Campus Id as a string** (e.g. `"1"`), not the campus name. A scanned card carries a campus name; resolve it to a Campus Id via the connect-time campus cache (`account_metadata['rockrms']['campuses']`) before writing it into a campus attribute.
- **Phone search needs digits only.** Strip all non-digits before the phone filter. An empty digit string means skip phone search entirely. The `PhoneNumbers/any(...)` filter runs server-side before `$select`, so trimming the result to `Id,FirstName,LastName,PrimaryAliasId` does not break the match.
- **No phone write on update (v1).** `PATCH /api/People` does not accept a phone field — phone lives in the PhoneNumbers sub-collection. Update writes first name / last name / email only. Phone is still used for matching and for filling workflow attributes.
- **Rock returns timestamps in LOCAL server time, not UTC** (this install is ~7h behind UTC). This does not affect the write path, but any timestamp you read back from a created record — or store in `rock_sync_last_attempted` from a Rock response — must be treated as Rock-local, not UTC, and stored as a plain `YYYY-MM-DDThh:mm:ss` string with no trailing `Z` and no offset.
- **Never overwrite with blanks.** On update, only include a field in the PATCH if the card actually has a value for it. A missing/empty card field must leave the existing Rock value alone.
- **The CareFlow never blocks.** Treat the push as fire-and-record: mark `'pending'` first, attempt, then record the outcome. Any exception leaves the card `'pending'` for the retry sweep — it must never propagate up and stall Connect Card creation or the CareFlow.

---

## Master Reference Index

Every API call used anywhere in this spec, with direct links. **Swagger** = the live API explorer on the install (`https://saltlight.rockcloud.com/api/docs`, then open the named controller). **Test Env doc** = the verified-curls document: [Rock RMS Test Environment doc](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit). Canonical spec hub: [rock-rms.html](https://saltlight-richard.github.io/saltlight-specs/specs/rock-rms.html).

Every row carries **two real hyperlinks** — the Swagger controller and the named verified curl — never plain text.

### General documentation
- Live API explorer (Swagger): https://saltlight.rockcloud.com/api/docs
- Rock developer hub: https://community.rockrms.com/developer
- Rock REST API overview: https://community.rockrms.com/developer/rock-api
- OData conventions used throughout: `$filter`, `$expand`, `$select`, `$orderby`, `$top`

### Endpoint index (by feature)

| Feature | What | Endpoint (+ key query params) | Method | Direct links |
|---|---|---|---|---|
| 1 | Permissions probe (does the key have People access?) | `/api/People/GetCurrentPerson` | GET | [Swagger › People](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Connection: permissions probe"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| 1 | Resolve Active RecordStatus GUID → install Id | `/api/DefinedValues?$filter=Guid eq guid'618F906C-C33D-4FA3-8AEF-E58CB7B63F1E'&$select=Id` | GET | [Swagger › DefinedValues](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Connection: resolve DefinedValue GUID"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| 1 | Cache active campuses at connect time | `/api/Campuses?$filter=IsActive eq true&$select=Id,Guid,Name,ShortCode` | GET | [Swagger › Campuses](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Campuses list"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| 1b | List the church's forms (WorkflowTypes) | `/api/WorkflowTypes?$filter=IsActive eq true&$select=Id,Guid,Name&$orderby=Name asc` | GET | [Swagger › WorkflowTypes](https://saltlight.rockcloud.com/api/docs) · [Test Env › "WorkflowTypes list"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| 1b | List a form's fields (Attributes) | `/api/Attributes?$filter=EntityTypeId eq 113 and EntityTypeQualifierColumn eq 'WorkflowTypeId' and EntityTypeQualifierValue eq '<typeId>'&$select=Id,Key,Name,FieldTypeId` (EntityTypeId 113 = the Workflow entity type — verified) | GET | [Swagger › Attributes](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Form fields (Attributes)"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| 1b | Resolve a field's dropdown options (DefinedValue picker) | `/api/AttributeQualifiers?$filter=AttributeId eq <id> and Key eq 'definedtypeguid'&$select=Value` → `/api/DefinedTypes?$filter=Guid eq guid'<g>'&$select=Id` → `/api/DefinedValues?$filter=DefinedTypeId eq <id>&$select=Id,Value&$orderby=Order asc` | GET | [Swagger › AttributeQualifiers](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Field values (DefinedValue)"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| 1b | Resolve a field's options (Value List field) | `/api/AttributeQualifiers?$filter=AttributeId eq <id> and Key eq 'values'&$select=Value` (pipe-delimited `Label,value` pairs) | GET | [Swagger › AttributeQualifiers](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Field values (Value List)"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| 2 | List Person tags for write-back dropdown | `/api/Tags?$filter=EntityTypeId eq 15&$select=Id,Guid,Name&$orderby=Name asc` | GET | [Swagger › Tags](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Tags list"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| 3 (Track 1) | Fetch People changed since cursor | `/api/People?$filter=ModifiedDateTime gt datetime'<cursor>'&$expand=PhoneNumbers&$orderby=ModifiedDateTime asc&$top=500` (filter/order/cursor advance ALL use ModifiedDateTime; never add `$select` — it strips PhoneNumbers) | GET | [Swagger › People](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Track 1: People poll"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| 3 (Track 2) | Fetch Completed workflow instances since cursor | `/api/Workflows?$filter=WorkflowTypeId eq <typeId> and Status eq 'Completed' and CreatedDateTime gt datetime'<cursor>'&$orderby=CreatedDateTime asc&$top=500` (filter/order/cursor advance ALL use CreatedDateTime; `Status eq 'Completed'` belongs IN the $filter — verified working on 18.1.0.0, do NOT filter Status in memory and do NOT use ModifiedDateTime here) | GET | [Swagger › Workflows](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Track 2 call 1: Workflows poll"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| 3 (Track 2) | Batch-fetch attribute values for those workflows | `/api/AttributeValues?$filter=(AttributeId eq 8281 or … or AttributeId eq 8286)&$select=AttributeId,EntityId,Value&$top=5000` (join to workflow Ids in memory by `EntityId == workflow Id`; do **not** filter by EntityId at scale) | GET | [Swagger › AttributeValues](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Track 2 call 2: AttributeValues batch"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| 4 | Search existing person by email | `/api/People?$filter=Email eq '<email>'&$top=5&$select=Id,FirstName,LastName,PrimaryAliasId` | GET | [Swagger › People](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Write-back: search by email"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| 4 | Search existing person by phone | `/api/People?$filter=PhoneNumbers/any(p: p/Number eq '<digits>')&$top=5&$select=Id,FirstName,LastName,PrimaryAliasId` | GET | [Swagger › People](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Write-back: search by phone (verify OData any)"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| 4 | Create new Rock person (returns bare integer Id) | `/api/People` (body: ConnectionStatusValueId, RecordTypeValueId, RecordStatusValueId, names, email) | POST | [Swagger › People](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Write-back: create person"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| 4 | Update existing Rock person (returns HTTP 204) | `/api/People/{id}` (only non-blank fields; never overwrite with blanks) | PATCH | [Swagger › People](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Write-back: update person"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| 4 | Read PrimaryAliasId for workflow association | `/api/People/{id}?$select=Id,Guid,PrimaryAliasId` (note: `$select` ignored on single-record GET — read defensively) | GET | [Swagger › People](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Write-back: get alias"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| 4 | Create a workflow instance in Rock (returns bare integer Id) | `/api/Workflows` (body: WorkflowTypeId, Name, InitiatorPersonAliasId, Status) | POST | [Swagger › Workflows](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Write-back: create workflow"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| 4 | Check for an existing AttributeValue row on the new workflow | `/api/AttributeValues?$filter=EntityId eq <wfId> and AttributeId eq <attrId>&$select=Id` | GET | [Swagger › AttributeValues](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Write-back: AV check"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| 4 | Create an AttributeValue (returns bare integer Id) | `/api/AttributeValues` (body: EntityId, AttributeId, Value) | POST | [Swagger › AttributeValues](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Write-back: AV create"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| 4 | Update an AttributeValue (returns HTTP 204) | `/api/AttributeValues/{id}` (body: Value) | PATCH | [Swagger › AttributeValues](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Write-back: AV update"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| 4 | Read a person's Guid before tagging | `/api/People/{id}?$select=Id,Guid` (TaggedItems needs the Guid, not the Id) | GET | [Swagger › People](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Write-back: get guid"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |
| 4 | Apply a tag to a person (EntityTypeId 15 REQUIRED) | `/api/TaggedItems` (body: TagId, EntityGuid, **EntityTypeId: 15**) | POST | [Swagger › TaggedItems](https://saltlight.rockcloud.com/api/docs) · [Test Env › "Write-back: apply tag"](https://docs.google.com/document/d/1nn6t0ELgdHmDGC7P7d3rf-nGXGkHsFyCUEkKVugaUTs/edit) |

### Test environment access

- **URL:** https://saltlight.rockcloud.com — Rock **18.1.0.0**.
- **Auth:** REST key passed as `Authorization-Token` header (the live key is in the Test Environment doc, not here). Service account is a RestUser in `RSR - Rock Administration` — if you ever get a 401, it is almost always a missing security role, not a bad key.
- **WorkflowType 24** = "First Time Guest - SaltLight". AttributeIds: **FirstName 8281, LastName 8282, Email 8283, Phone 8284, Campus 8285, InterestArea 8286**. InterestArea is Feature 2's trigger field, **not** one of the five mapped core fields. InterestArea options: Sunday Service, Youth Ministry, Kids Ministry, Small Groups.
- **Campuses:** 1 = Main, 2 = North.
- **Tag:** Id 1 = "Staff" (a Person tag, EntityTypeId 15).

**Loaded test data (created 2026-06-02, ready to pull):**
- **Track 1 — People 22 (PollerTest Alpha), 23 (Bravo), 24 (Charlie)** — each with a mobile phone.
- **Track 2 — Workflows 8 (Delta / Sunday Service), 9 (Echo / Youth Ministry), 10 (Foxtrot / Kids Ministry)** — Status = Completed, all 6 attributes populated.
- **Suggested poll cursor:** `2026-06-02T12:11:00` (Rock-local time, no trailing `Z`) — catches all 6 records.
- **Ignore:** People 20–21 (SpecTest Validation2026) are stale leftovers; Rock blocks `DELETE` on `/api/People` so they cannot be removed. (Older docs reference People 9–18 and Workflows 1–5 — fine as extra fixtures, but the 22–24 / 8–10 set is the canonical 2026-06-02 batch.)

### Portable GUIDs (identical on every Rock install — safe to hard-resolve at connect time)

```
Active RecordStatus:      618F906C-C33D-4FA3-8AEF-E58CB7B63F1E
Inactive RecordStatus:    1DAD99D5-41A9-4865-8366-F269902B80A4
Visitor ConnectionStatus: B91BA046-BC1E-400C-B85D-638C1F4E0CE2
Member ConnectionStatus:  41540783-D9EF-4C70-8F1D-C9E83D91ED5F
Mobile PhoneType:         407E7E45-7B2E-4FCD-9605-ECB1339F2453
Home PhoneType:           AA8732FB-2CEA-4C76-8D6D-6AAA2C6A4303
```
Resolve each GUID to its install-specific integer Id via `/api/DefinedValues?$filter=Guid eq guid'<GUID>'&$select=Id`. Note the **`guid'...'`** prefix is required — a bare quoted string returns HTTP 400. On the test install these resolve to: Active RecordStatus = 3, Visitor ConnectionStatus = 66, Mobile PhoneType = 12 (install-specific — never hardcode these integers, resolve from the GUID).

### Confirmed-broken endpoints (do NOT use)

| Endpoint | Result | Use instead |
|---|---|---|
| `/api/Utilities/GetRockSemanticVersion` | HTTP 404 on 18.1.0.0 | Don't depend on a version probe. |
| `PUT /api/Workflows/{id}/SetAttributeValue` | HTTP 404 | Create/update the AttributeValue row directly via `/api/AttributeValues` (GET to check, POST to create, PATCH to update). |
| `POST /api/Workflows/{id}/SaveAttributeValues` | HTTP 404 | Same as above. |
| `$expand=AttributeValues` on `/api/Workflows` | HTTP 400 always | Batch-fetch via `/api/AttributeValues` and join in memory. |
| `$expand=Campus` on `/api/People` | Silently ignored | Only scalar `PrimaryCampusId` is available (often null — campus lives on the Family Group); resolve via the campus cache. |
| `/api/v2/...` | HTTP 404 on self-hosted | Use `/api/{Entity}` OData v1 only. |

### Cross-cutting API facts (true everywhere — keep these in mind on every call)

- **All list endpoints return a bare array `[...]`**, not `{"value":[...]}` — verified across People, Workflows, AttributeValues, WorkflowTypes, Tags, Attributes, AttributeQualifiers, Campuses, DefinedValues, DefinedTypes on this install. Treat every list response as a bare array.
- **Rock returns and stores timestamps in LOCAL server time, not UTC.** A record created at 19:11 UTC comes back stamped 12:11 (this install runs ~7h behind UTC). Store every poll cursor as a plain `YYYY-MM-DDThh:mm:ss` string on Rock's clock — **no trailing `Z`, no offset** — so it drops straight into the `datetime'...'` filter. A UTC cursor lands in the future and silently returns nothing. (This is why the suggested test cursor is `2026-06-02T12:11:00`, Rock-local.)
- **`POST` to People, Workflows, and AttributeValues each return a bare integer** (the new row Id), not an object. Read the return value directly.
- **`PATCH` to `/api/People/{id}` and `/api/AttributeValues/{id}` return HTTP 204** with no body — success is "no exception," not a parsed response.
- **OData datetime filters MUST use `datetime'YYYY-MM-DDThh:mm:ss'`** — a bare ISO string returns HTTP 400.
- **GUID filters MUST use `guid'XXXXXXXX-...'`** — a bare quoted string returns HTTP 400.
- **`$select` silently strips `PhoneNumbers` from `$expand=PhoneNumbers`** — never combine them on the People poll.
- **`$select` is ignored on single-record GETs** (`/api/People/{id}`) — the full object comes back; read fields defensively.
- **CampusPicker AttributeValues store the Campus Id as a string** (`"1"`), not the name — resolve through the campus cache.
- **`POST /api/TaggedItems` requires `EntityTypeId: 15` for a Person** — omitting it returns HTTP 500. Rock allows duplicate TaggedItem rows (no 409); duplicates are harmless.
- **EntityTypeId 113 = the Workflow entity type** — verified in the Test Environment doc. Use it (not a guessed integer) when listing a form's Attributes.

### Poll schedule (Feature 1 setup, Feature 3, and the Storage map agree on this exact shape)

Stored at `account_metadata['rockrms']['poll_schedule']`:

```json
{
  "high_frequency_days": [5, 6],
  "high_frequency_start": "<service window start>",
  "high_frequency_end": "<service window end>",
  "low_frequency_interval_hours": 12,
  "timezone": "America/Chicago"
}
```

`high_frequency_days` `[5, 6]` = Saturday (5) and Sunday (6). Meaning: poll every 5 min on Sat/Sun during the service window; otherwise twice daily (`low_frequency_interval_hours: 12`). EventBridge fires every 5 min; a should-poll gate decides whether to actually poll. Per-church configurable.

### Storage map (where each feature's state lives)

| State | Location |
|---|---|
| Connection state (connected flag, domain, campus cache, status ids) | `account.account_metadata['rockrms']` |
| Credentials | AWS Secrets Manager `saltlight/rockrms/tokens/{env}/account-{account_id}` |
| Account-level form field mapping (Feature 1b) | `account_metadata['rockrms']['mapped_forms']` |
| Per-CareFlow Rock config (Feature 2) | `workflow.workflow_data['rockrms']` |
| Poll cursors + schedule (Feature 3) | `account_metadata['rockrms']['poll_cursor_track1' \| 'poll_cursor_track2' \| 'last_polled' \| 'poll_schedule']` |
| SaltLight ↔ Rock person link (Feature 4) | `person.person_metadata['rockrms_person_id']` |
| Routing key (domain) reserved for future webhook routing | `account_metadata['rockrms']['org_id']` (= the Rock domain; **not consumed by any v1 feature** — v1 ingestion is polling only) |
