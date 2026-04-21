# Automated warm-up framework (Bloomreach Engagement)

A reference framework + Jinja snippets for automating IP / domain warm-ups
inside Bloomreach Engagement (BRE) with as much logic as possible kept
in-platform.

This document is the long-form companion to the conversation that produced
it. It contains four sections:

1. [Critique of the original framework](#1-critique-of-the-original-framework)
2. [Recommended in-platform components](#2-recommended-in-platform-components)
3. [Snippets](#3-snippets)
4. [End-to-end flow](#4-end-to-end-flow)

The Jinja snippets themselves live in [`snippets/`](./snippets) so they can
be copy-pasted directly into the BRE asset library.

---

## 1. Critique of the original framework

The original framework proposed:

- A warm-up plan modelled as **events** with attributes for date, warm-up
  number, and per-domain target volume.
- A **metric** counting sent volume per domain.
- A **Jinja snippet** returning `true` if `domain_sent_volume <
  domain_target_volume AND killswitch inactive`.
- A **Jinja snippet** for activating a killswitch.
- An alert scenario notifying the delivery team on killswitch trip.

The direction is right, but several of the building blocks map awkwardly
to BRE primitives or have race conditions at scale:

### A. The plan should be a Catalog, not Events

Events are immutable, customer-scoped, append-only. The plan is none of
those — the delivery team needs to *edit* it during the warm-up. Use a
**Catalog**: free spreadsheet import/export, in-Jinja lookup via
`catalog_item(...)`, and edits are non-destructive.

### B. "Domain" almost certainly means "mailbox provider"

`gmail.com` and `googlemail.com` are one bucket; Microsoft is `outlook.com
+ hotmail.com + live.com + msn.com + ...`. Key the plan on a derived
`email_provider` attribute, not raw domain, otherwise you'll either
over-segment tiny domains or split Microsoft volume in half.

### C. A Jinja snippet cannot be a killswitch on its own

Jinja runs at render time. Returning `false` from a snippet renders an
empty template — it doesn't *stop* the send. Enforcement has to happen in
the **scenario** (Condition node), in a **campaign Audience filter**, or
in a **Frequency Policy**. The snippet is the *predicate*; the scenario
is the *gate*.

### D. The "sent volume per domain" metric has a measurement gap

BRE metrics are designed for analytics, not as a low-latency runtime
counter. There are three viable approaches:

1. **Build-time gating (recommended).** Each morning, build the day's
   audience per provider capped at `target_volume`, tag those customers
   with a 24h `warm_up_eligible_today_<provider>` flag. The gate snippet
   reads the flag — no runtime counter needed; you cannot physically
   exceed the target.
2. **Real-time aggregate + Condition node.** Define a per-customer
   aggregate and read it in scenarios. Works but the aggregate evaluates
   on the *processed* customer, not globally — so you actually need a
   "system customer" + webhook to bump it. Fiddly.
3. **Hourly metric + scenario rebuild.** Compute hourly, copy into a
   project-scoped catalog row, read from Jinja. High latency.

Option 1 is materially simpler and removes the race condition in (E).

### E. Race condition between behavioural and newsletter sends

With runtime counting, two behavioural sends to the same provider firing
in the same second both pass `< target` and you overshoot. Build-time
gating fixes this because the bound is on **audience size**, not a
counter.

### F. "Deliverability dropped" needs a precise trigger

Day 1 you might send 50 to Microsoft. One bounce = 2% bounce rate, which
trips almost any sensible threshold. The killswitch needs:

- A **minimum-volume floor** before thresholds apply (e.g. `>= 500` sends
  to that provider in the rolling window).
- **Separate** thresholds for hard bounce, soft bounce / deferral, spam
  complaint.
- A **rolling** window (last 6h or last 2 sends), not the warm-up-to-date
  total.
- **Per-provider** triggers, not a single global trigger.

### G. Define the volume target

ESPs care about *accepted* volume (sent minus hard-bounce minus dropped).
*Sent* is the right control knob (it's what you can throttle), but report
on *delivered* and adjust if list health is poor.

### H. Audience definition is missing

Who, specifically, gets today's warm-up send? Recommendation:
**most-engaged-first**, expanding to lower engagement tiers as days
progress. Days 1–3 most-engaged only; days 4–7 expand to mid-tier; from
day 8 include cold-but-consented. Without this rule the algorithm is
"random N from consented base" and will under-perform the manual process.

### I. End-of-day burst is a bad signal to ESPs

A single 23:00 dump is exactly the pattern ESPs throttle. Spread the
newsletter across 1–2h with randomised send time, or split into 4
batches.

### J. Model resets / holds explicitly

Add `is_held` and `held_until_date` fields on the plan catalog so the
delivery team flips one flag rather than editing many rows. Same for a
global `is_paused` and per-provider pauses.

### K. Auditability

Every killswitch trip and reset should fire a `warm_up_killswitch_changed`
event with `actor`, `reason`, `provider`, `previous_state`, `new_state`.
Otherwise post-mortems are guesswork.

### L. The third advantage in the original framework was empty

Suggested filling: *"It removes a class of human error — forgetting to
bump the attribute, mistyping the day number, or applying the wrong
audience."*

---

## 2. Recommended in-platform components

| Component | Type | Purpose |
|---|---|---|
| `email_provider` | Derived customer attribute | Maps `customer.email` domain to `google` / `microsoft` / `yahoo` / `apple` / `other`. Used for grouping and for plan lookups. |
| `warm_up_plan` | Catalog | Per-`(date, provider)` plan row. Columns: `id` (`YYYY-MM-DD_<provider>`), `date`, `provider`, `warm_up_day`, `target_volume`, `behavioural_max_pct`, `is_active`, `is_paused`, `plan_version`, `notes`. |
| `warm_up_settings` | Catalog | One row `global` plus one per provider. Columns: `killswitch_active`, `killswitch_tripped_at`, `killswitch_tripped_by`, `killswitch_reason`, `mode` (`live` / `shadow` / `complete`), `last_reset_at`, `last_reset_by`. |
| `warm_up_deliverability` | Catalog | One row per provider per day, populated hourly. Columns: `delivered_pct`, `hard_bounce_pct`, `soft_bounce_pct`, `complaint_pct`, `open_rate`, `click_rate`, `sent_count`. |
| `warm_up_blackout` | Catalog | `(date, provider, reason)` rows for blackout days. |
| `warm_up_eligible_today_<provider>` | Customer attribute (boolean) | Set by the morning audience-build scenario. Read by the gate snippet. |
| `wu_eligible_base` | Segment | Consented + no hard bounce ever + no complaint last 90d + active in last X months. |
| `wu_tier_a` / `wu_tier_b` / `wu_tier_c` | Segments | Engagement tiers, all subsets of `wu_eligible_base`. |
| `wu_can_send` | Snippet | Predicate: returns `"true"` / `"false"` for the current recipient. |
| `wu_killswitch_state` | Snippet | Read-only state reporter, used in alert templates and dashboards. |
| Morning audience-build | Scenario (scheduled 00:05) | Builds today's audience per provider, sets the `warm_up_eligible_today_<provider>` flag. |
| Hourly deliverability publish | Scenario (scheduled hourly) | Writes the latest metrics into `warm_up_deliverability`. |
| Killswitch evaluator | Scenario (event- or schedule-triggered) | Reads `warm_up_deliverability`, flips killswitch via webhook → catalog API. |
| Killswitch alert | Scenario (event-triggered on `warm_up_killswitch_changed`) | Notifies delivery team. |
| Daily digest | Scenario (scheduled 23:30) | Emails team planned vs sent vs delivered, killswitch events, tomorrow's plan. |
| Newsletter campaign | Campaign | Audience filter `wu_can_send == "true"`. |
| Behavioural scenarios | Scenarios | Welcome / abandon-cart / browse / category / search, each with a `wu_can_send == "true"` Condition node before the Email node. |

### A note on "killswitch activation" as a snippet

Setting state is a *write*. The in-platform options are:

1. **Scenario → Webhook → BRE Catalog API** updating the `warm_up_settings`
   row. Auditable, atomic, works with the snippets in this doc.
2. **Scenario → Update Customer Property** on a designated `system_warm_up`
   customer. No webhook, but the gate snippet has to look up a foreign
   customer on every render — more expensive at scale.

Default to option 1.

---

## 3. Snippets

The snippets live as standalone files so they can be copy-pasted into BRE.

- [`snippets/wu_can_send.jinja`](./snippets/wu_can_send.jinja)
- [`snippets/wu_killswitch_state.jinja`](./snippets/wu_killswitch_state.jinja)

### Wiring `wu_can_send` into sends

- **Newsletter campaign**: add an Audience filter
  `Snippet "wu_can_send" equals "true"`.
- **Behavioural scenario**: insert a Condition node before each Email node,
  branch on the same comparison.
- Combine with a **Frequency Policy** of *one warm-up touch per recipient
  per day* so behavioural + newsletter cannot stack.

### Catalog / aggregate accessor names

Bloomreach's Jinja accessors vary slightly by project. The snippets use
`catalog_item('<catalog_id>', '<row_id>')` and `customer.<attr>`, which
are the most common patterns. Swap to your project's actual function
names if they differ.

---

## 4. End-to-end flow

From "customer has given email volumes × health & domain" to "warm-up
finished".

### Phase 0 — One-time setup in the BRE project

1. Create the `email_provider` derived attribute.
2. Create the `warm_up_plan`, `warm_up_settings`, `warm_up_deliverability`,
   and `warm_up_blackout` catalogs.
3. Create the per-provider `warm_up_eligible_today_<provider>` customer
   attributes.
4. Create the two snippets (`wu_can_send`, `wu_killswitch_state`).
5. Create the `wu_eligible_base` segment and the engagement-tier segments
   (`wu_tier_a` / `b` / `c`).

### Phase 1 — Customer-supplied inputs (Day -3 to Day 0)

6. Receive total addressable volume per provider, list health, and
   constraints (deadline, blackouts, weekends).
7. Build the warm-up plan in the spreadsheet (existing template).
8. **Import the spreadsheet** into the `warm_up_plan` catalog.
   `is_active = true`, `is_paused = false`, `plan_version = "v1"`.
   Populate `warm_up_blackout` from the customer's blackout dates.
9. Initialise `warm_up_settings` rows: all `killswitch_active = false`,
   `mode = "shadow"` for the first 48h.
10. Configure killswitch thresholds (per-provider hard-bounce,
    soft-bounce / deferral, complaint, with min-volume floors).

### Phase 2 — Daily run loop (Day 1 onward)

11. **00:05 — Morning audience build (per provider).** For each provider
    with an active, unpaused plan row for today: pull `target_volume`,
    select that many customers from `wu_eligible_base` for that provider
    ordered by engagement tier (A → B → C) then last-engagement recency
    descending, set `warm_up_eligible_today_<provider> = "true"` on
    selected customers and `"false"` on everyone else. Track
    `warm_up_audience_built` event.
12. **All day — Behavioural sends fire** as customers trigger them. Each
    Email node sits behind a Condition `wu_can_send == "true"`. Track
    `warm_up_send_made` with `surface=behavioural`.
13. **End-of-day window (19:00–21:00) — Newsletter send.** Audience
    filter `wu_can_send == "true"` AND not received warm-up send in last
    24h (Frequency Policy). Spread over the window; split delivery in 4
    batches.
14. **Hourly — Deliverability metric publish** scenario writes the latest
    rolling-window metrics per provider into `warm_up_deliverability`.

### Phase 3 — Killswitch & exception handling (continuous)

15. **Killswitch evaluator** (hourly + on bounce/complaint events): for
    each provider read latest `warm_up_deliverability` row; if
    `sent_count >= min_volume_floor` AND any threshold breached, POST to
    catalog API: `warm_up_settings.<provider>.killswitch_active = true`,
    populate `tripped_at`, `tripped_by = auto_<reason>`,
    `reason = "<metric>=<value>"`. Track
    `warm_up_killswitch_changed`.
16. **Alert scenario** triggered by `warm_up_killswitch_changed` →
    email + Slack/webhook to delivery team with provider, reason, current
    numbers, dashboard link.
17. **Manual reset** (delivery team): edits the relevant `warm_up_settings`
    row to set `killswitch_active = false`, populate `last_reset_at`,
    `last_reset_by`. Optionally lowers `target_volume` or sets
    `is_paused = true` for the next N days. Tracks
    `warm_up_killswitch_changed` with `tripped_by =
    "manual_reset:<email>"`.
18. **Hold pattern** (point 7 of the original framework): set
    `is_paused = true` on the affected provider's plan rows for the
    desired flat days, OR duplicate yesterday's row forward N times so
    volume holds flat instead of growing.

### Phase 4 — Reviews

19. **Daily 23:30 digest** scenario emails the team: planned vs sent vs
    delivered vs bounced vs complained per provider, killswitch events
    in last 24h, tomorrow's plan.
20. **Weekly review** (manual, data-driven from the same catalogs).

### Phase 5 — Completion

21. When the last `warm_up_plan` row's date has passed AND the last 7
    days of deliverability are within target ranges, a scenario sets
    `warm_up_settings.global.mode = "complete"`.
22. `wu_can_send` short-circuits to `"true"` for all providers when
    `mode = "complete"` (the gate disengages; killswitch still
    enforceable).
23. Final summary email to customer + delivery team. Snapshot catalogs
    for post-mortem.
24. Behavioural scenarios continue unchanged — they're now just BAU.

---

## Appendix — Improvements not in the original framework

In rough priority order:

1. **Engagement-tiered audience composition** (most-engaged-first,
   expanding by day).
2. **Provider-aware throttling** (hourly curves per provider, carried in
   the plan catalog).
3. **Send-time send-window guard** (don't send at 03:00 if the
   audience-build runs late).
4. **Consent + suppression hygiene** checked at gate time, not just
   upstream.
5. **Per-provider deliverability monitoring catalog** (the
   `warm_up_deliverability` table above) gives the killswitch a stable
   read source and powers a dashboard for free.
6. **Two-tier killswitch**: soft pause (skip rest of today, resume
   tomorrow) vs hard kill (until human resets).
7. **Dry-run / shadow mode** flag making `wu_can_send` always return
   `false` while logging "would-have-sent" events. Use for the first
   24–48h of a new project.
8. **Rebuild-on-edit**: when the team edits a plan row mid-day, fire an
   event-driven scenario that rebuilds today's eligibility flags for
   that provider.
9. **Plan version pinning** on every gate event for clean attribution
   when the team adjusts mid-warm-up.
10. **Holiday / blackout calendar** as a separate catalog.
11. **Capacity headroom** (`behavioural_max_pct`) so behavioural sends
    can't consume the entire day's volume and starve the newsletter.
12. **Daily summary email** at 23:30 to the delivery team.
13. **Exit criteria** explicitly modelled (`mode = "complete"` flag).
14. **Per-IP, not just per-domain** warm-up if you have multiple sending
    IPs — add an IP dimension to the plan and round-robin in the
    audience build.
