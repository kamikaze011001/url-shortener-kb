---
status: accepted
date: 2026-09-04
---

# Statistics are grouped by UTC day

`click_events.occurred_at` is a `timestamptz`, so grouping Clicks "by day" requires
choosing a timezone. The choice is **UTC**, and it is visible to the user: an Owner in
GMT+7 who clicks their own Link at 06:00 local sees that Click land on the *previous*
day.

The frontend states the zone next to the date range rather than hiding it, because a
number that is off by a day without explanation is worse than a number with a label.

## Considered options

**The Owner's timezone.** The obvious user-friendly answer, and the reason it is rejected
is not effort: the same Link would then report **different daily numbers to different
people**. An Owner sharing a screenshot with a colleague in another zone produces two
charts that disagree, and neither is wrong. Support conversations about analytics become
unanswerable without first establishing whose clock is being discussed.

**The timezone of the browser making the request,** resolved per request. Same problem as
above, plus the numbers change when the Owner travels.

**A timezone stored per Owner, chosen at registration.** This is the correct long-term
answer and is deferred, not rejected. It keeps one authoritative zone per account, so the
numbers are stable and explainable. It needs a column, a setting screen, and a decision
about what happens to historical charts when the setting changes — none of which fit
before the demo.

## Consequences

- **Daily buckets are the same for every reader**, including anyone querying Postgres
  directly. A number in the dashboard and a number from `psql` agree, which is not true
  under any per-viewer scheme.
- **The offset is a real usability cost**, not a rounding detail. For an Owner at GMT+7,
  seven hours of every local day are attributed to the day before.
- **Migrating to per-Owner zones later re-buckets history.** Existing charts will change
  shape the day that ships. That is acceptable because these counts are analytics and not
  an audit log — [FR-5](../01-requirements.md) already says they are approximate by
  design — but it must be a stated change, not a silent one.
- **The range is capped at 366 days** and the daily series is gap-filled server-side. A
  day with no Clicks is a zero, not missing data; a chart that omits it draws a line
  straight from Monday to Friday as though Tuesday never happened.
