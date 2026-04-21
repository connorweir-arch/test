# Automated Warm-Up Framework (Bloomreach Engagement)

A framework for warming up new sending domains and IPs faster, with less manual intervention, using as many in-platform tools as possible.

This document covers:

1. A constructive critique of the proposed framework
2. Two Jinja snippets (`can_send_to_domain` and `killswitch`)
3. Suggested improvements not in the original draft
4. End-to-end step-by-step flow from "customer hands over volumes" to "warm-up complete"

---

## 1. Constructive critique

The framework is directionally strong — moving from manually-managed attributes + system scenarios to event-driven plans is the right call. Below are the points worth pushing back on before you build it.

### 1.1 Conceptual / design risks

- **"Domain volume target" conflates two different things.** A warm-up plan is usually expressed per-IP × per-receiving-domain (Gmail, Yahoo, Microsoft, etc.), but your spec only talks about "domain volume target". If you have multiple sending IPs / sub-domains being warmed in parallel, you need a composite key on the plan event: `sending_identity` × `receiving_domain` × `warm_up_date`. Otherwise you'll either over-send on shared infra or under-utilise a fast-warming IP.
- **No definition of "domain".** Decide up front whether you bucket by exact MX (`gmail.com`, `googlemail.com`, `outlook.com`, `hotmail.com`, …) or by ESP family (Google, Microsoft, Yahoo/AOL, Apple, B2B/Other). The plan, the metric, and the snippet must agree. ESP-family bucketing is what the major IP-warming guides use and it matches how reputation is actually calculated.
- **"Behavioural sends fill the volume, newsletter fills the rest" is great — but order matters.** If the newsletter is scheduled at end-of-day and behaviourals run during the day, you must guarantee that behaviourals also respect the same `can_send_to_domain` snippet. Otherwise behaviourals can blow through the cap before the newsletter even queues. Your draft says this implicitly; make it an explicit rule and a code-review checklist item.
- **Killswitch granularity.** A single global killswitch is blunt. In practice you will want at least three levels: global, per sending identity (IP/sub-domain), and per receiving-domain family. A single bad day at Microsoft shouldn't pause Google warming.
- **"Killswitch on deliverability drop" needs a precise trigger.** "Deliverability dropped" is not a metric. Decide which thresholds trip it: hard-bounce %, complaint %, deferral/4xx %, blocklist hit, or a *change* relative to a 7-day rolling baseline. Absolute thresholds are simpler; rolling baselines catch creeping degradation. Pick one and write it down.
- **Killswitch should *latch*.** A momentary dip should not auto-recover the next minute and resume sending. The snippet should stay tripped until a human resets it (and ideally records *why*). That's how this is implemented in the snippet below.
- **No "ramp down" pathway.** Plans tend to assume monotonic growth. Real warm-ups frequently need to hold or step backwards (your point 7). The plan event must support a non-increasing target on day N+1, and the operations runbook must allow re-uploading just the affected rows without disturbing the rest of the plan.
- **Audience suppression order.** Make sure your warm-up audience is intersected with: consented, not unsubscribed, not on the suppression list, not a hard-bounced address, not in any other concurrent test/promo. Otherwise you'll re-introduce previously bounced addresses as "fresh volume" and tank reputation immediately.
- **Reciprocity with seed/probe sends.** Add seedlist sends (Inbox Monitor / GlockApps / 250ok-style) and a small monitoring panel to *every* day's batch. They cost ~nothing in volume but give you placement data the ESP postmaster tools won't.
- **Engagement-first selection is missing from the spec.** The single biggest lever during warm-up is *who* you send to, not how many. Sort the audience by engagement (last open / click recency, predicted open propensity), then truncate to the day's target. Don't pick at random or by registration date.
- **No cohort isolation.** During warm-up, do not co-mingle the warming domain with already-warmed traffic, or with low-engagement re-engagement campaigns. Your framework doesn't say how that isolation is enforced.
- **"Newsletter scheduled at end of day" can hurt placement at consumer ESPs.** Send timing should be aligned to the recipient's local timezone window where engagement is highest, not to the operations team's day. End-of-day-UTC sends arrive overnight in APAC and miss the morning-open window in EMEA. Use BRE's send-time optimisation if available, and if not, fall back to per-region scheduled batches.

### 1.2 Operational risks

- **No recovery SLA.** When the killswitch trips, what's the response time? Who resets it? Is there an on-call rota? This is a control system; the human-in-the-loop part needs an SLA or the killswitch becomes a foot-gun.
- **No audit trail requirement.** Every killswitch trip and reset should write an event with: who/what tripped it, the metrics at the time, who reset it, and the rationale. Otherwise you'll repeat the same incident in three weeks.
- **No "dry-run" or canary mode.** First time you run this on a real customer there will be bugs. Add a flag on the plan event (e.g. `mode: 'dry_run' | 'canary' | 'live'`) so the snippet can render placement decisions to a log event without actually sending.
- **No back-pressure from the ESP.** If Gmail starts deferring, you'll keep stuffing more into the queue because your `sent` metric counts accepted-by-MTA, not delivered. Either include 4xx-deferral rate in the killswitch, or switch the metric from "sent" to "delivered in the last 24h".
- **No definition of "warm-up complete".** Add explicit exit criteria: e.g., 7 consecutive days at full target volume with bounce < X%, complaint < Y%, inbox placement > Z%. Without exit criteria, warm-ups quietly drag on.
- **Spreadsheet as source of truth is a smell.** It's the right starting point because your ops team lives there, but the plan event in BRE should become the source of truth post-import, and the snippet should never read the spreadsheet. Treat the spreadsheet as an import artefact, not as live config.
- **Pause/resume semantics.** Pausing the warm-up should be a single action that bumps the killswitch and records "paused at day N". Resuming should *not* simply uncork sending: it should advance the plan dates by the pause length, otherwise day-N volume goes out on day N+pause and breaks the gradient.

### 1.3 Things that aren't actually wrong, but are easy to get wrong

- "Strong baseline of performance each day" is true *only* if behavioural triggers are themselves well-tuned. A misfiring abandoned-cart with 0.4% open will hurt, not help. Quality-gate the behavioural templates before they're allowed in the warm-up.
- Personalising subject lines is fine but be wary of spammy tokens (emoji, all-caps, urgency words); ESPs grade subject lines too.
- Don't use the same Jinja snippet to make the gate decision *and* to render the email body. Keep gating logic in audience filters / event conditions so it's testable in isolation.

---

## 2. Jinja snippets

> Both snippets live in this repo for review under [`snippets/warm-up/`](../../snippets/warm-up). Bloomreach Jinja prints values rather than returning them, so the snippets emit the literal strings `TRUE` / `FALSE`. In an audience filter, compare against the string (`snippet:can_send_to_domain == 'TRUE'`).

### 2.1 `can_send_to_domain.jinja`

This is the per-recipient gate. Use it on every send in the warm-up programme — newsletter and behaviourals — as the final filter.

It does three things, in order:

1. Reads the global `killswitch_active` flag and short-circuits to `FALSE` if it's tripped.
2. Looks up the day's plan row for the recipient's domain.
3. Compares today's sent-volume aggregate against the plan target.

Anything missing or ambiguous returns `FALSE` (fail-safe).

```62:90:snippets/warm-up/can_send_to_domain.jinja
        {%- set today = now() | date('%Y-%m-%d') -%}
        {%- set plan = events('warm_up_plan',
                              filter={'domain': domain,
                                      'warm_up_date': today},
                              order='desc',
                              count=1) -%}

        {%- if plan | length == 0 -%}
FALSE
        {%- else -%}
            {%- set target = plan[0].properties.domain_volume_target | int(0) -%}

            {# ---- 3. Lookup volume already sent to this domain today ----- #}
            {%- set sent = aggregate('warm_up_sent_today',
                                     filter={'domain': domain}) | int(0) -%}

            {%- if target > 0 and sent < target -%}
TRUE
            {%- else -%}
FALSE
            {%- endif -%}
        {%- endif -%}
```

Full file: [`snippets/warm-up/can_send_to_domain.jinja`](../../snippets/warm-up/can_send_to_domain.jinja)

### 2.2 `killswitch.jinja`

The killswitch is a *latched* gate: once tripped it stays tripped for `LATCH_HOURS` (default 24h) or until a human resets it. It checks, in priority order:

1. Manual override (`killswitch_manual`) — instant kill from the delivery team.
2. Latched state — has a `killswitch_tripped` event been written within `LATCH_HOURS`?
3. Live deliverability thresholds — hard-bounce, complaint and delivered rates per domain.

```44:78:snippets/warm-up/killswitch.jinja
{%- if manual in [true, 'true', 'TRUE', 1, '1'] -%}
TRUE
{%- else -%}

    {# ---- 2. Latched: was tripped recently and not yet reset? ----------- #}
    {%- set last_trip = events('killswitch_tripped',
                               order='desc',
                               count=1) -%}
    {%- set latched = false -%}
    {%- if last_trip | length > 0 -%}
        {%- set hours_since = ((now() - last_trip[0].timestamp) / 3600) | int(99999) -%}
        {%- if hours_since < LATCH_HOURS -%}
            {%- set latched = true -%}
        {%- endif -%}
    {%- endif -%}

    {%- if latched -%}
TRUE
    {%- else -%}

        {# ---- 3. Live deliverability thresholds (per domain) ------------ #}
        {%- set bounce    = aggregate('bounce_rate_24h',
                                      filter={'domain': domain}) | float(0) -%}
        {%- set complaint = aggregate('complaint_rate_24h',
                                      filter={'domain': domain}) | float(0) -%}
        {%- set delivered = aggregate('delivered_rate_24h',
                                      filter={'domain': domain}) | float(1) -%}

        {%- if bounce    > HARD_BOUNCE_MAX
           or complaint > COMPLAINT_MAX
           or delivered < DELIVERED_MIN -%}
TRUE
        {%- else -%}
FALSE
        {%- endif -%}
```

Full file: [`snippets/warm-up/killswitch.jinja`](../../snippets/warm-up/killswitch.jinja)

### 2.3 Implementation notes & caveats

- **Function names are illustrative.** BRE Jinja exposes `customer`, project attributes, events, and aggregates, but the exact helper names (`events(...)`, `aggregate(...)`) and signatures vary by project. Map them to your tenant's actual primitives before deploying. The *shape* of the logic is what matters.
- **Are snippets the right unit?** Yes for the gate (`can_send_to_domain`) because it's evaluated per-recipient inside a campaign filter. The killswitch is *also* fine as a snippet, but a stronger pattern is to materialise its output into a project-level attribute (`is_killswitch_tripped`) updated by a scenario every minute. That way you read a cheap attribute on every send rather than re-evaluating thresholds per recipient. The snippet then becomes the scenario's expression rather than a per-send computation.
- **Cost.** `aggregate(...)` calls in a per-recipient snippet are not free. If your warm-up batches are large, materialise both `sent_today_by_domain` and the killswitch state into project / catalog attributes refreshed on a schedule (e.g. every 1–5 minutes), and have the snippets read those attributes instead. The snippet code stays the same; only the data source changes.
- **Fail-safe defaults.** Every unknown / missing branch returns `FALSE` (do not send) and `TRUE` (kill is on). When in doubt, do less.
- **Comparison style.** I used string `'TRUE'/'FALSE'` because BRE filters are easier to reason about with string equality. If your filter syntax supports boolean snippets natively, change the literals to `true`/`false`.

---

## 3. Suggested improvements not in the original draft

These are items I'd add to the "required components" or "approach" sections.

### 3.1 Plan & data model
- **Composite plan key:** `sending_identity` × `receiving_domain_group` × `warm_up_date` × `target` × `mode` (`live`/`canary`/`dry_run`).
- **Receiving-domain grouping table:** explicit mapping (Gmail family, Microsoft family, Yahoo/AOL family, Apple, Other). Drives both the plan and the snippet's bucketing.
- **Plan versioning:** every spreadsheet import writes events with a `plan_version` attribute. Re-imports supersede by `(domain, date, plan_version)` so you can roll back a bad change.
- **Suppression intersection:** documented at the audience level, not in the snippet. The snippet trusts that the audience is already clean.

### 3.2 Audience selection
- **Engagement-ranked truncation:** order candidates by predicted-engagement / recency, then take top-N to fill the day's target. Don't let the snippet pick by random order of evaluation.
- **Cohort isolation:** warming traffic uses dedicated From-address / sub-domain; never overlapped with already-warmed BAU sends in the same batch.
- **Seedlist injection:** add 50–200 seed addresses per ESP family per day; their placement is your early-warning system *before* bounce/complaint metrics move.

### 3.3 Killswitch & monitoring
- **Three-tier killswitch:** global, per sending identity, per receiving-domain group. Snippet reads the most-specific level first.
- **Trip on rate-of-change, not just absolute thresholds:** e.g. complaint rate is *3× the 7-day rolling mean* and *> 0.05%*.
- **Defer-rate input:** include 4xx deferrals in the trip logic — Gmail/Microsoft will slow you down before they bounce you.
- **Reset workflow:** killswitch reset is itself an event with `reset_by`, `reason`, `new_targets[]`. Auditable.
- **Alerting:** scenario fires a Slack/email/PagerDuty webhook on trip *and* on reset, with the metrics snapshot attached.

### 3.4 Sending behaviour
- **Daily batch cap *and* hourly drip cap.** ESPs penalise spiky senders. Spread the day's volume across the highest-engagement local-time windows; don't fire it all in one batch at end-of-day.
- **Send-time optimisation** per recipient where available.
- **Subject-line guardrails:** automated check against a small banned-token list before contextual personalisation runs.
- **Behavioural-quality gate:** behavioural templates must pass a baseline (e.g. >25% open, <0.1% complaint) on warmed traffic before being allowed in the warm-up programme.

### 3.5 Reporting
- **Daily warm-up dashboard:** per `(sending_identity, receiving_domain_group, warm_up_day)` show target, sent, delivered, bounce%, complaint%, open%, click%, killswitch state.
- **Plan-vs-actual chart** with a deviation alert when actual < 80% of target for two consecutive days (under-utilisation is a problem too).
- **Exit-criteria check:** an automated "warm-up complete?" view rather than a human eyeballing the chart.

### 3.6 Process
- **Runbook with named owners:** "killswitch tripped" page, "Microsoft deferring" page, "ramp-down" page, "pause/resume" page.
- **Pre-warm-up checklist:** SPF/DKIM/DMARC aligned, BIMI optional, `p=quarantine`/`reject` plan agreed, postmaster tools (Google, Microsoft SNDS/JMRP, Yahoo CFL) signed up *before* day 1.
- **Post-warm-up review:** every warm-up ends with a 30-min retro that updates the templated volumes spreadsheet for the *next* customer.

---

## 4. End-to-end flow

From "customer has handed over email volumes × health × domain" to "warm-up complete".

### Phase A — Pre-flight (before day 1)

1. **Authentication & infrastructure check**
   - SPF, DKIM, DMARC alignment on the warming sub-domain.
   - PTR records on dedicated IPs.
   - `p=none` on DMARC for the warm-up window; agree the migration to `quarantine`/`reject` after completion.
   - Sub-domain isolated from any already-warmed traffic.
2. **Postmaster enrolments**
   - Google Postmaster Tools for the sending domain.
   - Microsoft SNDS + JMRP for each sending IP.
   - Yahoo CFL.
   - Seedlist provider (Inbox Monitor / GlockApps / equivalent).
3. **List hygiene**
   - Run the customer's lists through validation (e.g. NeverBounce, Kickbox).
   - Remove role addresses, hard bounces, spam-traps, and unengaged > 12-month addresses.
   - Apply existing suppression list.
4. **Build the plan**
   - From the templated volumes × the customer's audience × ESP-family share, generate the warm-up plan spreadsheet.
   - Columns: `sending_identity`, `receiving_domain_group`, `warm_up_date`, `warm_up_day_number`, `domain_volume_target`, `mode`.
5. **Import the plan as events**
   - Upload spreadsheet → `warm_up_plan` events in BRE, one per row.
   - Tag with `plan_version = v1`.
6. **Configure aggregates / metrics**
   - `warm_up_sent_today` per receiving-domain-group (resets daily).
   - `bounce_rate_24h`, `complaint_rate_24h`, `delivered_rate_24h`, `defer_rate_24h` — all per domain group.
7. **Deploy snippets and killswitch**
   - Deploy `can_send_to_domain` and `killswitch` (or the materialised-attribute equivalents).
   - Set `killswitch_manual = false`.
   - Verify the alert scenario fires on a manually-tripped test.
8. **Build / verify campaigns**
   - Warm-up newsletter campaign with `can_send_to_domain == 'TRUE'` filter.
   - Behavioural sends (welcome, abandoned cart, browse, category, search) with the same filter.
   - All campaigns intersected with consent / suppression.
9. **Canary day**
   - Run day 0 in `mode = 'canary'`: ~10% of day-1 target, audited end-to-end. No real warm-up credit; this is rehearsal.

### Phase B — Daily warm-up loop (day 1 → day N)

For each day:

1. **Morning: health check (automated dashboard, human eyes)**
   - Postmaster reputation, seed-list inbox placement, bounce/complaint/defer rates from the previous 24h.
   - Confirm killswitch is `FALSE`.
2. **Behavioural traffic flows during the day**
   - Behavioural triggers (welcome, abandon cart, browse, category, search) fire as usual; `can_send_to_domain` gates each one.
   - Volume to each domain group accumulates against the day's target.
3. **End-of-day window: warm-up newsletter**
   - Audience: engaged subscribers, ranked by predicted engagement, intersected with `can_send_to_domain == 'TRUE'`.
   - The snippet ensures the newsletter only takes the *remaining* volume up to the target.
4. **Continuous: killswitch evaluation**
   - The killswitch scenario re-evaluates thresholds every 1–5 minutes.
   - On trip: writes `killswitch_tripped` event, fires alert, all sends gate to `FALSE` immediately.
5. **End-of-day: automated reporting**
   - Plan-vs-actual report posted to delivery channel.
   - Exit-criteria check: are we sustaining target with bounce < X%, complaint < Y%, inbox > Z%?

### Phase C — Exception handling (any day)

- **Killswitch tripped**
  1. Alert hits the delivery team channel with metrics snapshot.
  2. On-call investigates root cause (bad template, list issue, ESP incident, blocklist hit).
  3. Decision tree:
     - *Transient ESP issue:* hold 24h, then reset killswitch with the same plan.
     - *Reputation impact (mild):* reset killswitch, issue a downward step-back on the affected domain group's targets for 2–3 days, then resume the curve.
     - *Reputation impact (severe):* reset to a holding pattern (flat low volume) for the affected domain group for 5–7 days; investigate sending behaviour; only then resume.
  4. Reset action writes a `killswitch_reset` event with `reset_by`, `reason`, optional `new_targets[]`. The plan import re-runs only for affected `(domain, date)` rows.
- **Customer-requested pause**
  1. Set `killswitch_manual = true`.
  2. When resuming, *shift* the remaining plan dates forward by the pause duration (don't skip).

### Phase D — Completion

1. **Exit criteria met:** N consecutive days (typically 5–7) at 100% of full target with bounce, complaint, and inbox-placement metrics inside band, across every receiving-domain group.
2. **Migrate**
   - DMARC policy moved from `p=none` → `p=quarantine` → `p=reject` (per the customer's pre-agreed plan).
   - Disable warm-up newsletter; behavioural triggers continue without the gate.
   - Remove the `can_send_to_domain` filter from BAU campaigns (or leave it in place returning `TRUE` so re-warming is a single-flag affair).
3. **Retro & template update**
   - Capture the actual curve, the tripped-killswitch incidents, and any manual interventions.
   - Update the templated volumes spreadsheet for the next customer.
4. **Hand-off to BAU**
   - Delivery team retains ownership of the killswitch and dashboards.
   - Document the new sending identity in the customer's BAU runbook.

---

## 5. Acceptance checklist

Use this as the definition-of-done for shipping the framework.

- [ ] `warm_up_plan` event schema published, with example import file.
- [ ] `warm_up_sent_today` aggregate live and resetting daily.
- [ ] `bounce_rate_24h`, `complaint_rate_24h`, `delivered_rate_24h`, `defer_rate_24h` aggregates live, per domain group.
- [ ] `can_send_to_domain` snippet deployed and unit-tested with: missing email, missing plan row, target=0, sent>=target, killswitch on/off.
- [ ] `killswitch` snippet (or materialised attribute equivalent) deployed; manual trip + auto-reset behaviour verified.
- [ ] Alert scenario fires on trip *and* reset, with metrics snapshot.
- [ ] Warm-up newsletter campaign uses the gate filter; behavioural sends use the same filter.
- [ ] Audience selection uses engagement-ranked truncation, not arbitrary order.
- [ ] Seedlist enrolled and reporting daily.
- [ ] Postmaster tools (Google / Microsoft / Yahoo) enrolled.
- [ ] Pause/resume runbook written and rehearsed.
- [ ] Exit criteria automated in the daily report.
