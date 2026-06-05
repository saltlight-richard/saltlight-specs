# Multi-Selection Routing — Backend Spec for Kenneth

**Replaces all prior versions. This is the single source of truth.**
Audience: Kenneth (backend engineer). This spec covers **internal SaltLight routing behavior** — how SaltLight decides which leader(s) a connect card goes to when a guest makes one or more selections. It is **not** an integration; nothing here talks to an external system. (For how a submission's trigger selection becomes the card selection this routing consumes, see the CareFlow 6-step editor Step 6 selection-mapping reference in the appendix.)

### How to read this document

This spec is deliberately code-free. It tells you **what** the routing must do and **where** the data lives — never **how** to write it. You own all the code. The **data model comes first** (the Data Model and API Changes section), before any rule, because every routing rule in this document depends on two new columns on the `workflow` table (`priority_rank` and `always_fire`). Read the Data Model and API Changes section, then the rules will make sense.

Every rule follows the same shape so you can skim:

1. **Breadcrumb** — a 🧭 locator at the top of the section so you always know where you are.
2. **The Rule** — one or two paragraphs of plain English: what this rule does.
3. **User story** — one sentence: who wants what, and why.
4. **User journey** — the concrete step-by-step narrative, with one worked example using the canonical seed ranks (Accepted Christ = 1, New Guest = 2) plus the illustrative third CareFlow (Baptism Interest = 3).
5. **Acceptance criteria** — testable, observable `- [ ]` checkboxes. If you can't watch it happen in the database, a log, or the leader's thread, it isn't done.
6. **What the backend touches** — plain-English map of which `workflow` columns, which `/workflow` endpoints, and which tables the rule reads or writes. No logic.

Code fences appear in this document **only** to show data shapes — an example stored column blob, or a column table. They never contain logic. If you see a fence, it is data, not an instruction to copy code.

### Breadcrumb / nav legend

Every major section opens with a breadcrumb so you always know where you are:

> 🧭 Multi-Selection Routing › {Section Name}

- **🧭** = you are here (top-of-section locator).
- A checkbox `- [ ]` in Acceptance criteria = a thing a tester (Richard or you) must be able to observe — in the database, a log, or a leader's message thread.
- **Rule N** sections (Rules 1-4) each describe one routing behavior and reuse the shape above.
- **EC-1 .. EC-7** (Edge Cases) are the edge cases — Kenneth-requested; all seven are kept.
- Code fences are **data only** (a column table or a stored config blob) or a single navigational decision diagram; they never contain code/logic to copy. If you see a fence, it is data or a navigation aid.

## Data Model and API Changes

> 🧭 Multi-Selection Routing › Data Model and API Changes

This section comes first because every routing rule in this document depends on these two columns existing on the `workflow` table. Read this before any rule.

### The two new columns on `workflow`

Two columns are added to the `workflow` table. Nothing else in the schema changes.

| Column | Type | Default | Notes |
|--------|------|---------|-------|
| `priority_rank` | INTEGER | NULL | Positive integers `1..N`. Rank 1 = highest priority; the **lowest number wins**. Must be **NON-NULL** when `workflow_active = true` — a CareFlow cannot be activated without a rank. Unique among **active, ranked** CareFlows within one account (see constraint below). No cap (a church with 20 CareFlows uses ranks 1..20). |
| `always_fire` | BOOLEAN | FALSE | When `true`, this CareFlow fires **alongside** the priority winner whenever its selection matches on a multi-selection card, regardless of its rank position. |

### Uniqueness constraint

`priority_rank` is unique only among **active, ranked** CareFlows, scoped per account. Enforce with a **partial unique index**:

- Index on `(account_id, priority_rank)`
- Condition: `WHERE workflow_active = true AND priority_rank IS NOT NULL`

Inactive CareFlows and NULL ranks never collide — only an active, ranked CareFlow can conflict with another active, ranked CareFlow in the same account.

### Seed ranks for new accounts

When a new account is provisioned, two CareFlows are seeded with ranks:

- **Accepted Christ** = rank `1`
- **New Guest** = rank `2`

### `/workflow` API changes

Three changes to SaltLight's internal `/workflow` endpoints. The error strings below are exact — return them verbatim.

- **`POST /workflow` and `PUT /workflow`** — if `workflow_active = true` AND `priority_rank` IS NULL, return **HTTP 400** with the message:
  `A priority rank is required before activating a CareFlow.`
- **`PUT /workflow` (rank change)** — if the new rank is already in use by another **active** CareFlow in the same account, return **HTTP 409** with the message:
  `Rank [N] is already assigned to [CareFlow Name]. Please choose a different rank.`
- **`GET /workflow/list`** — every record in the response includes the `priority_rank` and `always_fire` fields.

### Stored shape (data only)

How the two new columns appear on a single stored `workflow` row (Accepted Christ, active, rank 1):

```json
{
  "workflow_active": true,
  "priority_rank": 1,
  "always_fire": false
}
```

### What the schema does NOT touch

The `interaction` table is **unchanged**. No columns are added or altered there. The only schema change in this entire feature is the two columns above — `priority_rank` and `always_fire` — added solely to the `workflow` table.

## Overview

> 🧭 Multi-Selection Routing › Overview / Mental Model

When a guest submits a connect card, SaltLight decides which leader(s) receive a follow-up assignment based on how many selections the guest made and which active CareFlows those selections match. This section is the mental model for that decision; Rules 1-4 (plus the No-Match and Catch-All fall-through) are the precise implementation of it, and they all depend on the two `workflow` columns defined in the Data Model and API Changes section (`priority_rank`, `always_fire`).

### The routing principle

Three invariants hold on every routing path:

- **No guest is ever silently dropped.** Every confirmed card resolves to a destination — a matching CareFlow, the catch-all, or an admin notification. There is no path where a guest receives nothing.
- **No leader is ever notified twice for the same guest.** Matched CareFlows are grouped by their primary assigned leader (`workflow_to_person_primary`) so a leader assigned to two or more matched CareFlows receives exactly one two-message sequence — even though an `interaction` record is still created for every matched CareFlow.
- **The system resolves without an admin.** An admin is only pulled in as the last resort: nothing matched and no catch-all is configured. Every other decision is made automatically.

### The universal two-message sequence

Every leader who receives a guest — regardless of routing path — gets the same two-message sequence, in this order:

- **Message 1 — Context only:** the guest's full name, *every* selection the guest made on the card (not only the one that matched this CareFlow), any prayer request, and any open-field content. No suggested reply. No options.
- **Message 2 — AI-generated suggested reply + options:** the AI synthesizes a reply from the full guest context (all selections + prayer request + open-field), written in the leader's voice and generated fresh — never pulled verbatim from the CareFlow template. Options, in this exact order and wording:
  - `SEND` — deliver the draft as-is.
  - `NO` — redraft.
  - `@` — write your own message (sent directly to the guest).

For a deduplicated leader (see Rule 4), this is one Message 1 covering the full combined context, followed by one Message 2 synthesized from that combined context using the priority winner's CareFlow as the primary voice/tone reference.

### Routing decision at a glance

```
                    ┌─────────────────────────────┐
                    │  How many selections did     │
                    │  the guest make?             │
                    └─────────────────────────────┘
                                 │
            ┌────────────────────┴─────────────────────┐
            │                                           │
   exactly one selection                       2+ selections
   (matching one CareFlow)                              │
            │                                           │
   direct fire (Rule 1)                  ranking ALWAYS applies (Rule 2)
   priority NOT evaluated                          │
            │                          lowest priority_rank = winner
   ┌────────┴────────┐                             +
   │  TERMINATES     │             any always_fire CareFlow whose selection
   │  here — never   │             matched fires alongside (Rule 3)
   │  re-tested for  │                             │
   │  no-match       │             group matched CareFlows by primary leader
   └─────────────────┘             → one two-message sequence per leader (Rule 4)
                                                   │
                                        nothing matched?
                                                   │
                                    ┌──────────────┴──────────────┐
                                    │                              │
                            catch-all configured            no catch-all
                                    │                              │
                      guest silently enrolled →         admin notified with
                      two-message sequence fires        SaltLight's proposed action
```

Read top to bottom: **single → direct** (terminates); **multi → priority winner (+ always-fire) → leader dedup → catch-all → admin**. The "nothing matched?" node and the catch-all / admin-notification fall-through below it sit only on the multi-selection path: it fires when no active CareFlow matches on that path. A single selection that already matched a CareFlow fires directly per Rule 1 and is never re-tested for no-match — its branch terminates before the convergence. (The single-selection path can also reach no-match in its own right when a lone selection matches nothing; that follows the same fall-through described in No-Match and Catch-All.)

## Rule 1 — Single Selection

> 🧭 Multi-Selection Routing › Rule 1 — Single Selection

### The Rule

When a guest makes exactly one selection and that selection matches exactly one CareFlow, that CareFlow fires directly. Priority ranking is not evaluated — there is no contest to resolve, so `priority_rank` plays no part in single-selection routing.

The matched CareFlow produces exactly one `interaction` record (status: `initial`), and the assigned leader receives the universal two-message sequence (Message 1 = context only; Message 2 = AI-generated suggested reply + `SEND` / `NO` / `@`).

### User Story

As a church leader, when a guest submits a connect card with one selection that matches my CareFlow, I want to receive two messages — first a context message telling me who the guest is and what they selected, then a second message with a suggested reply I can send, redraft, or replace — so I can follow up with full situational awareness and no guesswork.

### User Journey

1. SaltLight receives a confirmed card for a guest who made one selection: "I'm new here."
2. The system matches "I'm new here" to the New Guest CareFlow (seed rank 2). Because only one selection was made, `priority_rank` is not evaluated.
3. The system creates one `interaction` record (New Guest CareFlow, status: `initial`).
4. The first message is scheduled for delivery within the CareFlow's configured send window.
5. The assigned leader receives Message 1: guest's full name, the selection ("I'm new here"), prayer request (if any), open-field content (if any). No suggested reply, no options.
6. The leader receives Message 2: an AI-generated suggested reply written in the leader's voice, followed by the options `SEND` / `NO` / `@`.
   - `SEND` delivers the draft as-is.
   - `NO` redrafts.
   - `@` lets the leader write their own message, sent directly to the guest.
7. The leader replies with `SEND`, `NO`, or `@`, and the guest receives the message.

> **Worked example.** A guest checks only "I'm new here" on their card. New Guest (rank 2) is the single match, so it fires directly — rank 2 is irrelevant because no other CareFlow is in contention. One `interaction` is created for New Guest, and the assigned leader (e.g. Pastor Mike) receives one Message 1 + one Message 2. Note that New Guest is rank 2, never rank 1.

### Acceptance Criteria

- [ ] A guest with exactly one selection matching one CareFlow triggers that CareFlow to fire directly.
- [ ] Priority ranking is not evaluated for single-selection guests.
- [ ] One `interaction` record is created for the matched CareFlow (status: `initial`).
- [ ] The leader receives Message 1 and Message 2 as two separate messages (never combined).
- [ ] Message 1 includes: guest's full name, the selection made, prayer request (if any), open-field content (if any).
- [ ] Message 2 includes an AI-generated reply — not the verbatim CareFlow template — and the options `SEND` / `NO` / `@`.

### What the backend touches

- **Reads** the confirmed card's single selection and matches it against active CareFlows in the `workflow` table.
- **Does not read** `priority_rank` or `always_fire` — single-selection routing bypasses ranking entirely.
- **Writes** one row to the `interaction` table (status: `initial`) for the matched CareFlow. The `interaction` schema is unchanged.
- **Resolves** the assigned leader via `workflow_to_person_primary`, who then receives the two-message sequence.

## Rule 2 — Multi-Selection and Priority Ranking

> 🧭 Multi-Selection Routing › Rule 2 — Multi-Selection and Priority Ranking

### The Rule

If a guest makes more than one selection, priority ranking always applies. Among the active CareFlows whose mapped selections match any of the guest's selections, the one with the lowest `priority_rank` number fires as the priority winner (rank 1 = highest priority; lowest number wins). Matching CareFlows that did not win priority — and do not have the always-fire flag — do not fire. The guest is handled by the priority winner, and exactly one `interaction` record (status `initial`) is created for that winner.

If none of the guest's selections match any configured CareFlow and no always-fire CareFlow applies, the catch-all CareFlow fires and logs the guest. This is the safety net for unique or unexpected connect cards — selections that fall outside whatever the church has currently configured. No card ever comes in without somewhere to go. If no catch-all is configured and nothing matches, the admin is notified with a proposed action.

Priority ranking depends on the `priority_rank` rules established in the Data Model and API Changes section:

- Every active CareFlow must have a unique priority rank before it can go active. A CareFlow without a rank cannot be activated.
- Ranks are positive integers (1, 2, 3 … N). Rank 1 is the highest priority; the lowest number wins.
- No two **active, ranked** CareFlows in the same account can share the same rank number. Inactive CareFlows and NULL ranks never collide.
- There is no cap — a church with 20 CareFlows will have ranks 1 through 20.
- New accounts are pre-populated with: Accepted Christ at rank 1, New Guest at rank 2.

### User Story

As a church leader whose CareFlow is the highest-priority match for a guest who checked multiple boxes, I want to receive one two-message sequence that gives me context on everything the guest selected — not just the selection that matched my CareFlow — so that I can have a fully informed conversation with them.

### User Journey

1. SaltLight receives a confirmed card for a guest who made two selections: "I'm new here" and "Interested in baptism."
2. The system finds two matching CareFlows: New Guest (rank 2) and Baptism Interest (rank 3). Neither has always-fire enabled.
3. New Guest wins, because it has the lowest rank number among the matches. Only New Guest fires. Baptism Interest does not create an `interaction` record.
4. The system creates one `interaction` record for the New Guest CareFlow (status `initial`).
5. Pastor Mike (New Guest primary leader) receives Message 1: the guest's full name, all selections ("I'm new here" and "Interested in baptism"), and the prayer request if any.
6. Pastor Mike receives Message 2: an AI-generated suggested reply synthesizing both selections, followed by `SEND` / `NO` / `@`.
7. The guest receives the follow-up from Pastor Mike. No one from Baptism Interest is notified.

### Acceptance Criteria

- [ ] A guest with two or more selections triggers priority-ranking logic.
- [ ] A CareFlow cannot be set to active if `priority_rank` is null. The API returns HTTP 400 with the exact message: `A priority rank is required before activating a CareFlow.`
- [ ] No two active CareFlows in the same account can share the same `priority_rank`. The API returns HTTP 409 with the exact message: `Rank [N] is already assigned to [CareFlow Name]. Please choose a different rank.`
- [ ] `priority_rank` uniqueness is enforced at the database level by a partial unique index on `(account_id, priority_rank) WHERE workflow_active = true AND priority_rank IS NOT NULL`. Inactive CareFlows and NULL ranks never collide.
- [ ] New accounts are pre-populated with Accepted Christ at rank 1 and New Guest at rank 2.
- [ ] The system evaluates all active CareFlows whose mapped selections match any of the guest's selections.
- [ ] The matching CareFlow with the lowest `priority_rank` fires as the priority winner.
- [ ] CareFlows that matched but did not win priority — and do not have `always_fire = true` — do not fire. The guest is still handled by the priority winner.
- [ ] If none of the guest's selections match any configured CareFlow and no always-fire CareFlow applies, the catch-all CareFlow fires and creates an `interaction` record. No card is ever left without a destination.
- [ ] If no catch-all is configured and no CareFlow matches, the admin is notified with a proposed action.
- [ ] One `interaction` record is created for the priority winner.
- [ ] The priority winner's primary leader receives Message 1 and Message 2.
- [ ] Message 1 includes ALL of the guest's selections — not just the one that matched this CareFlow — plus the prayer request and open-field content.
- [ ] `GET /workflow/list` returns `priority_rank` and `always_fire` for each workflow.

### What the backend touches

- Reads `workflow.priority_rank` and `workflow.always_fire` for every active CareFlow in the account to determine the priority winner; reads `workflow_active` to scope candidates.
- Reads `workflow_to_person_primary` to identify the winner's primary leader for the two-message sequence.
- Writes one `interaction` record (status `initial`) for the priority winner; the `interaction` table schema is unchanged.
- `POST /workflow` and `PUT /workflow` validate that `priority_rank` is non-null when `workflow_active = true` (400); `PUT /workflow` additionally rejects a changed rank already in use by another active CareFlow (409). The partial unique index backs the 409 at the database level. `GET /workflow/list` returns `priority_rank` and `always_fire` on every record.

## Rule 3 — Always-Fire Flag

> 🧭 Multi-Selection Routing › Rule 3 — Always-Fire Flag

### The Rule

A boolean flag on each CareFlow (`always_fire`, default `false`). When enabled, that CareFlow fires alongside the priority winner whenever its selection is matched on a multi-selection card — regardless of its rank position.

This is intended for CareFlows where the church wants guaranteed notification no matter which CareFlow wins priority. SaltLight displays advisory copy near this setting: "Enabling this means this leader will always be notified when their selection is matched, even when another CareFlow takes priority. We recommend keeping this off unless you have a specific reason to notify multiple leaders."

### User Story

As a church leader whose CareFlow has the always-fire flag enabled, I want to be notified whenever my selection is matched — even when another CareFlow has higher priority — because my role requires me to always follow up with guests in my area regardless of what other leaders are doing.

### User Journey

1. SaltLight receives a confirmed card for a guest who made two selections: "Accepted Christ" and "I have a guest services question."
2. System finds two matching CareFlows: Accepted Christ (rank 1), Guest Services (rank 4, always-fire enabled).
3. Accepted Christ wins priority. Guest Services also fires due to the always-fire flag.
4. Accepted Christ is assigned to Pastor Sarah. Guest Services is assigned to the Guest Services leader. Different leaders — no deduplication needed.
5. System creates two `interaction` records (one per CareFlow).
6. Pastor Sarah receives Message 1 + Message 2 (full context of both selections).
7. Guest Services leader receives Message 1 + Message 2 (full context of both selections).
8. Guest receives follow-up from whichever leader acts first.

### Acceptance Criteria

- [ ] A CareFlow with `always_fire = true` fires alongside the priority winner whenever its selection is matched on a multi-selection card.
- [ ] An `interaction` record is created for each always-fire CareFlow that fires.
- [ ] The always-fire CareFlow's assigned leader receives Message 1 and Message 2.
- [ ] Always-fire CareFlows that did NOT match any of the guest's selections do not fire.
- [ ] `always_fire` defaults to `false` on new CareFlow creation.
- [ ] If the always-fire CareFlow is also the priority winner, it fires once — the flag has no additional effect on the winning CareFlow.

### What the backend touches

Reads the `always_fire` and `priority_rank` columns on `workflow` for every CareFlow whose selection matched the card. After Rule 2 resolves the priority winner, any matched CareFlow with `always_fire = true` is added to the fire set alongside the winner. Writes one `interaction` record (status `initial`) per always-fire CareFlow that fires, and resolves each firing CareFlow's primary leader via `workflow_to_person_primary` to send the two-message sequence (Message 1 = full guest context, including every selection the guest made; Message 2 = AI-synthesized suggested reply drawn from the full guest context, with options SEND / NO / @). No new endpoints; this rule consumes the columns added in the Data Model and API Changes section. The `interaction` table schema is unchanged.

## Rule 4 — Leader Deduplication

> 🧭 Multi-Selection Routing › Rule 4 — Leader Deduplication

### The Rule

Before any messages are sent, the system groups all matched CareFlows (the priority winner plus any always-fire matches) by their primary assigned leader. The dedup key is `workflow_to_person_primary`. If two or more matched CareFlows share the same primary leader, that leader receives exactly one two-message sequence — not one per CareFlow.

Interaction records are still created for every matched CareFlow. Deduplication affects only how many message sequences a leader receives; it never reduces the `interaction` records written for tracking.

### User Story

As a church leader assigned to multiple CareFlows, when a guest's selections match more than one of my CareFlows, I want to receive only one set of messages — not one per CareFlow — so I am not confused by duplicate notifications about the same guest.

### User Story — Guest Perspective

As a guest who checks multiple boxes on a connect card, I want to receive follow-up from the right leader(s) and have that leader know everything I indicated on my card — so my follow-up feels personal and I don't need to repeat myself.

### User Journey

> **Note:** This dedup example deliberately assigns the Accepted Christ CareFlow to Pastor Mike (instead of Pastor Sarah, who owns it elsewhere in this spec) to construct the shared-leader case — both matched CareFlows must share one primary leader for deduplication to apply.

1. SaltLight receives a confirmed card for a guest who made two selections: "Accepted Christ" and "I have a guest services question."
2. Both Accepted Christ (rank 1) and Guest Services (rank 4, always-fire enabled) match.
3. Both CareFlows are assigned to Pastor Mike — the same primary leader (`workflow_to_person_primary`).
4. The system detects the shared leader and deduplicates by that key.
5. The system creates two `interaction` records (one per matched CareFlow) for tracking purposes.
6. Pastor Mike receives one Message 1: the full context of both selections.
7. Pastor Mike receives one Message 2: an AI-generated reply synthesized from the combined context, using the priority winner's CareFlow (Accepted Christ, rank 1) as the primary voice and tone reference.
8. Pastor Mike is not notified twice.

### Acceptance Criteria

- [ ] If the priority winner and one or more always-fire CareFlows share the same primary assigned leader, that leader receives exactly one Message 1 and one Message 2.
- [ ] Deduplication is based on the `workflow_to_person_primary` leader assignment.
- [ ] The single Message 1 includes the full context of all the guest's selections.
- [ ] The single Message 2 is AI-generated from the combined context, using the priority winner's CareFlow as the primary voice reference.
- [ ] Interaction records are created for all matched CareFlows even when the messages are deduplicated.

### What the backend touches

Reads the `workflow_to_person_primary` leader assignment for each matched CareFlow to group by primary leader, and reads `priority_rank` to identify the winner whose CareFlow serves as the primary voice reference for the combined Message 2. Writes one `interaction` record (status `initial`) per matched CareFlow — deduplication suppresses duplicate message sequences only, never the `interaction` records.

## No-Match and Catch-All

> 🧭 Multi-Selection Routing › No-Match and Catch-All

### The Behavior

When a guest's card is processed and **no active CareFlow matches any of the guest's selections**, routing falls back to a no-match path so that every card still reaches a destination and the system resolves without dropping the guest. There are two outcomes, depending on whether a catch-all CareFlow has been configured for the account.

If a catch-all CareFlow is configured, the guest is **silently enrolled** in the catch-all and the assigned leader receives the standard two-message sequence — identical in format to every other routed path. If no catch-all is configured, the **admin** is notified with SaltLight's proposed action and the ability to confirm or adjust it.

### User Story

As an admin, I want unmatched guests to be routed to a catch-all CareFlow (or surfaced to me when there is no catch-all) so that no guest is ever silently dropped and every card has a destination.

### User Journey

1. A guest submits a card with one or more selections (or with content that yields no matchable selection).
2. The system evaluates the active CareFlows for the account and finds that **none** of them match any of the guest's selections.
3. **Catch-all configured:** The guest is silently enrolled in the catch-all CareFlow. The catch-all's primary leader receives the two-message sequence (Message 1 = full guest context; Message 2 = AI-generated suggested reply with the options `SEND` / `NO` / `@`). No admin action is required.
4. **No catch-all configured:** No leader is enrolled automatically. Instead, the admin receives a notification containing SaltLight's proposed action, which the admin can confirm or adjust.

**Worked example.** A guest selects only a custom option that does not map to Accepted Christ (rank 1), New Guest (rank 2), or Baptism Interest (rank 3), and no other active CareFlow matches. If the church has set up a catch-all CareFlow, the guest is silently enrolled in it and its leader gets the two-message sequence. If the church has no catch-all, the admin is notified with SaltLight's proposed action to confirm or adjust.

### Acceptance Criteria

- [ ] If no active CareFlow matches and a catch-all is configured, the guest is silently enrolled in the catch-all and the two-message sequence fires.
- [ ] If no active CareFlow matches and no catch-all is configured, the admin is notified with SaltLight's proposed action.
- [ ] When the catch-all fires, the catch-all's primary leader receives the standard two-message sequence (Message 1 context only; Message 2 AI-generated reply with `SEND` / `NO` / `@`).
- [ ] When the admin is notified, the notification includes a proposed action the admin can confirm or adjust.

### What the backend touches

This path reads the account's active CareFlows in the `workflow` table to determine that nothing matches. On the catch-all branch it enrolls the guest in the configured catch-all CareFlow and creates one `interaction` record (status: `initial`) for the catch-all's primary leader (`workflow_to_person_primary`). On the no-catch-all branch it creates no automatic enrollment and instead routes an admin notification. No new columns are read or written beyond those defined in the Data Model and API Changes section; the `interaction` table schema is unchanged.

## Edge Cases

> 🧭 Multi-Selection Routing › Edge Cases

These are the boundary conditions Kenneth should design around explicitly. Each edge case states the situation and the required behavior. None of them introduces new routing logic — they pin down how the rules above resolve when inputs are messy, in-flight, or defensive. All seven (EC-1 .. EC-7) are load-bearing; none may be dropped.

### EC-1 — Unmatched selection alongside matched selections

A guest checks a selection that matches no CareFlow at the same time as one or more selections that do match.

The system routes on the matching selections only. The unmatched selection still appears in the Message 1 context (Message 1 always lists every selection the guest made), but it does not affect which CareFlow fires.

### EC-2 — Only always-fire CareFlows match

The only matching CareFlows are always-fire CareFlows; no non-always-fire CareFlow matches.

The always-fire CareFlow with the lowest `priority_rank` number among the matches is treated as the priority winner. Every other matching always-fire CareFlow also fires alongside it. (Example: if Guest Services (always-fire) is the only match, it becomes the winner; if two always-fire CareFlows match, the lower-ranked one wins and the other fires alongside.)

### EC-3 — Admin reorders ranks after cards already processed

An admin changes `priority_rank` values after guest cards have already been routed.

Rank changes apply to future routing decisions only. Guests already in-flight are not re-routed. (For example, reordering Baptism Interest from rank 3 to rank 5 affects only cards processed after the change.)

### EC-4 — CareFlow deactivated between card confirmation and routing execution

A CareFlow is set inactive (`workflow_active = false`) in the window between card confirmation and routing execution.

That CareFlow is skipped. If it was the only match, fall back to the catch-all CareFlow, or to admin notification if no catch-all is configured.

### EC-5 — All matching CareFlows assigned to the same leader

Every matched CareFlow shares the same primary assigned leader (`workflow_to_person_primary`).

An `interaction` record is created for all matched CareFlows. The leader receives one Message 1 and one Message 2 covering the full combined context. (This is Rule 4 — leader deduplication — in practice.)

### EC-6 — Card with only open-field content and no checkbox selections

A guest submits a card containing only open-field content, with no checkbox selections.

There are no selections to route on, so the card is treated as no-match: the catch-all CareFlow if one is configured, otherwise admin notification.

### EC-7 — Rank collision bypasses the database constraint (defensive)

Two active, ranked CareFlows in the same account somehow end up sharing a `priority_rank`.

The partial unique index on `(account_id, priority_rank) WHERE workflow_active = true AND priority_rank IS NOT NULL` prevents this from occurring. As a defensive fallback, if a collision somehow does occur, the system fires the CareFlow with the alphabetically earlier name, logs an error, and alerts the admin:

```
Routing conflict detected — two CareFlows share rank [N]. Please resolve this in Settings.
```

### Acceptance Criteria

- [ ] An unmatched selection submitted alongside matched selections appears in Message 1 context but does not change which CareFlow fires (EC-1).
- [ ] When only always-fire CareFlows match, the lowest-ranked one is the winner and every other matching always-fire CareFlow also fires (EC-2).
- [ ] A rank reorder applied after cards are processed affects only future routing; in-flight guests are not re-routed (EC-3).
- [ ] A CareFlow deactivated before routing execution is skipped; if it was the only match, routing falls back to catch-all or admin notification (EC-4).
- [ ] When all matched CareFlows share one leader, `interaction` records are created for all matched CareFlows and the leader receives exactly one Message 1 + one Message 2 over the combined context (EC-5).
- [ ] A card with only open-field content and no checkbox selections is treated as no-match (catch-all if configured, else admin notification) (EC-6).
- [ ] The partial unique index blocks duplicate active ranks; if a collision is somehow present, the alphabetically earlier CareFlow fires, an error is logged, and the admin receives the exact alert string above (EC-7).

### What the backend touches

These edge cases read the `workflow` columns `priority_rank`, `always_fire`, and `workflow_active` (plus the partial unique index over `(account_id, priority_rank)`), and the `workflow_to_person_primary` primary leader assignment for the shared-leader case. They create `interaction` records (status `initial`) for every matched CareFlow exactly as the main rules do — no schema beyond the two new `workflow` columns is involved, and the `interaction` table is unchanged.

## What Does Not Change

> 🧭 Multi-Selection Routing › What Does Not Change

This feature adds two columns to the `workflow` table (`priority_rank`, `always_fire`) and a priority-resolution layer on top of routing. It does not alter any of the existing behavior listed below. If a change here appears to touch one of these, that is a bug.

- Two-message sequence **format** — Message 1 = context only; Message 2 = AI-generated reply + `SEND` / `NO` / `@`.
- CareFlow configuration (the 6-step editor).
- CareFlow timing, reminders, fallback escalation, and handoffs.
- The `interaction` table schema — unchanged. No columns are added or altered on `interaction`.
- The two new columns (`priority_rank`, `always_fire`) are added **only** to the `workflow` table; no other table changes.
- Single-selection routing — does not use priority ranking.
- Catch-all and no-match admin-notification behavior.
- CareFlow changes do not affect guests already in-flight.

## Reference appendix

> 🧭 Multi-Selection Routing › Reference appendix

This appendix is a consolidated lookup. It introduces no new behavior — every entry restates a fact already established in the Data Model and API Changes section and the rule sections. If anything here appears to disagree with an earlier section, the earlier section governs.

### Data-model recap

Multi-selection routing adds exactly **two columns to one table** (`workflow`). Nothing else in the schema changes — in particular, the `interaction` table is untouched.

| Column | Table | Type | Default | Notes |
|---|---|---|---|---|
| `priority_rank` | `workflow` | INTEGER, nullable | `NULL` | Positive integers `1..N`. Rank 1 = highest priority; **lowest number wins**. Must be NON-NULL when `workflow_active = true` (a CareFlow cannot be activated without a rank). UNIQUE among **active, ranked** CareFlows in one account, enforced by a partial unique index on `(account_id, priority_rank) WHERE workflow_active = true AND priority_rank IS NOT NULL`. Inactive CareFlows and NULL ranks never collide. No cap. |
| `always_fire` | `workflow` | BOOLEAN | `FALSE` | When TRUE, this CareFlow fires alongside the priority winner whenever its selection matches on a multi-selection card, regardless of rank position. |

Seed ranks for new accounts: **Accepted Christ = rank 1, New Guest = rank 2**. (Illustrative third CareFlow used in examples: Baptism Interest = rank 3.)

### `/workflow` API-change index

These are SaltLight's internal `/workflow` endpoints. There is no external API surface for this feature.

| Endpoint | Change | Error string (exact) |
|---|---|---|
| `POST /workflow` | If `workflow_active = true` and `priority_rank` is NULL, reject with HTTP 400. | `A priority rank is required before activating a CareFlow.` |
| `PUT /workflow` | If `workflow_active = true` and `priority_rank` is NULL, reject with HTTP 400. | `A priority rank is required before activating a CareFlow.` |
| `PUT /workflow` (rank change) | If the new rank is already used by another active CareFlow in the same account, reject with HTTP 409. | `Rank [N] is already assigned to [CareFlow Name]. Please choose a different rank.` |
| `GET /workflow/list` | Response includes `priority_rank` and `always_fire` on every record. | — |

One additional defensive error string is emitted by the routing path itself (see EC-7), not by a `/workflow` endpoint:

`Routing conflict detected — two CareFlows share rank [N]. Please resolve this in Settings.`

### Glossary

- **CareFlow** — A configured guest-care workflow defined in the 6-step editor, with a triggering selection and a primary assigned leader. A row in the `workflow` table.
- **Priority rank** (`priority_rank`) — The positive integer that orders active CareFlows for multi-selection routing. Rank 1 is highest; the **lowest** number wins. Required when a CareFlow is active; unique among active, ranked CareFlows in an account.
- **Always-fire flag** (`always_fire`) — A boolean on a CareFlow. When TRUE, the CareFlow fires alongside the priority winner whenever its selection matches on a multi-selection card, regardless of rank. If the always-fire CareFlow is also the winner, it fires once (the flag adds nothing extra). Always-fire CareFlows whose selections did not match do not fire.
- **Catch-all CareFlow** — The CareFlow a guest is silently enrolled in when no active CareFlow matches any selection. When configured, the two-message sequence fires for it. When no catch-all is configured and nothing matches, the admin is notified with SaltLight's proposed action instead.
- **Interaction record** — A row in the `interaction` table (status `initial`) created for every matched CareFlow that fires. One per matched CareFlow — including each always-fire CareFlow that fires and each CareFlow under a deduped leader. Deduplication reduces leader messages, never interaction records.
- **Two-message sequence** — The universal pair of messages a routed leader receives. Message 1 is context only (guest's full name, every selection the guest made, prayer request if any, open-field content if any; no suggested reply, no options). Message 2 is the AI-generated suggested reply written in the leader's voice, synthesized fresh from the full guest context, with options in this exact order and wording: `SEND` / `NO` / `@` (`SEND` = deliver as-is, `NO` = redraft, `@` = write your own, sent directly to the guest).
- **Primary leader** (`workflow_to_person_primary`) — The primary leader assignment on a CareFlow, configured in the 6-step editor. It is the deduplication key in Rule 4: a leader assigned to two or more matched CareFlows receives exactly one two-message sequence.

### Related specs

These are the only related-spec references for this document. There are no external API links.

- **CareFlow 6-step editor — Step 6 (selection mapping)** — how a submission's trigger selection becomes the card selection that routing consumes: Step 6 of the 6-step editor is where a CareFlow's triggering selection is mapped, and that selection is what this routing matches against. This is the internal selection-mapping reference the intro points to (referenced, not duplicated here).
- **CareFlow 6-step editor — Step 6 (leader assignment)** — where a CareFlow's primary leader (`workflow_to_person_primary`) is configured (referenced, not duplicated here).
- **Two-message guest-thread behavior** — Message 1 (context) / Message 2 (AI draft + `SEND` / `NO` / `@`) lives in SaltLight's guest-thread behavior.

### Storage map

Which table and column each rule reads or writes. Routing is read-mostly: it reads `workflow` to decide what fires, then writes one `interaction` per matched CareFlow. The two new `workflow` columns are written only through the `/workflow` editor endpoints, never by the routing path.

| Rule / behavior | Reads | Writes |
|---|---|---|
| Rule 1 — Single Selection | `workflow.workflow_active`, `workflow.workflow_data` (matching) | One `interaction` (status `initial`) for the matched CareFlow |
| Rule 2 — Multi-Selection & Priority Ranking | `workflow.priority_rank`, `workflow.workflow_active`, `workflow.workflow_data` | One `interaction` for the priority winner |
| Rule 3 — Always-Fire Flag | `workflow.always_fire`, `workflow.priority_rank`, `workflow.workflow_data` | One `interaction` per always-fire CareFlow that fires |
| Rule 4 — Leader Deduplication | `workflow_to_person_primary` (dedup key), `workflow.priority_rank` (identifies winner for the combined Message 2 voice reference) | One `interaction` per matched CareFlow (records always created; only leader messages are deduped) |
| No-Match / Catch-All | `workflow` (no active match found) | One `interaction` for the catch-all when configured; otherwise admin notification (no `interaction`) |
| Activation / rank edit | `workflow.priority_rank`, `workflow.workflow_active` (validation + partial unique index) | `workflow.priority_rank`, `workflow.always_fire` via `POST` / `PUT /workflow` |

The `interaction` table schema is unchanged — only the row creations described above use it.