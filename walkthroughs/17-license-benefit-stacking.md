# The License Count That Doubled Overnight — Stacking, Not a Gap

## TL;DR

Right after redeeming an annual Microsoft partner benefits package, the tenant's aggregate license counts for two SKUs abruptly doubled — one went from 35 to 70, another from 10 to 15 — alongside a batch of "N subscriptions will expire soon" warnings. The obvious read was either a licensing gap about to open up, or an overlap that needed cleaning up. Neither was right. Cross-referencing two different Microsoft views of the same tenant against each other showed the real mechanism: each year's benefit redemption creates a brand-new license batch stacked on top of the previous year's, rather than renewing or merging it. Nothing was broken, but there's no self-service way to clean it up, and it recurs — a little worse — every year.

## Purpose of this Document

A case study and reference runbook for the problem summarised in the TL;DR above.

It is intentionally written to:

- Give the cross-check procedure that separates benign stacking from a real coverage gap
- Keep that procedure separate from the first, wrong theory
- Document the stacking as an annual, expected event, so next year's identical-looking alarm is recognized in five minutes rather than re-investigated

## Environment

- Microsoft 365 licensing
- Microsoft Partner Center — annual Partner Success benefits redemption, "Your Products" summary
- Microsoft Entra ID — aggregate license/SKU view
- Microsoft 365 admin center — "N subscriptions will expire soon" notifications

---

## Correct End-to-End Process (Authoritative)

Run this whenever license counts jump or "expiring soon" warnings appear right after a benefits redemption.

### 1. Note the Two Signals

Two things landed close together, right after that year's Partner Success benefits redemption:

- Entra ID's aggregate license view showed SKU counts that had roughly doubled versus before the redemption.
- The admin center was sending "N subscriptions will expire soon" notifications for what looked like a meaningful chunk of licenses.

Read together, they suggest a specific, alarming story: a batch of licenses is about to lapse, and the doubled count is masking a coverage gap that will become visible the moment the expiring batch drops off. Don't act on that story yet — see [Issue 1](#issue-1--first-theory-a-transient-handover-overlap).

### 2. Pull the Aggregate View From Entra ID

**Entra ID's aggregate license/SKU view** — a single rolled-up count per SKU, with no indication of how many separate underlying orders make up that count.

### 3. Pull the Per-Order Breakdown From Partner Center

**The per-order breakdown from Partner Center / the "Your Products" summary** — lists each benefit redemption as its own distinct order, each with its own start and end date.

### 4. Cross-Reference the Two Views Side by Side

These two views of the same tenant don't usually get looked at side by side. Line them up and check:

- whether the current year's redemption is a **new, separate order**, additive to the previous year's, rather than a renewal or replacement
- which order the "expiring soon" warning actually refers to

In this case the current year's redemption had created an entirely new, separate license order, additive to the previous year's. The Entra aggregate view adds these orders together with no visual distinction — so from that view alone, a stacked total looks identical to a real increase in purchased licenses. See [Issue 2](#issue-2--aggregate-view-makes-stacking-look-like-growth).

### 5. Decide Whether the Expiring Order Is Load-Bearing

The expiry warnings were genuine — but they were warning about the *old* order's natural end date. That was never a threat to actual coverage, since the new order was already active and additive, not a replacement waiting to take over.

**If this all happens → nothing is broken; the old order's expiry is an artifact to expect, not a coverage risk.**

### 6. Record It as an Annual Pattern

There's no self-service tool to merge stacked batches from different years into one clean order — see [Issue 3](#issue-3--no-self-service-way-to-merge-stacked-orders). Rather than trying to force a one-time fix, document the pattern itself as an annual, expected event.

---

## Issues Encountered in This Case

### Issue 1 — First theory: a transient handover overlap

**Observed:** doubled aggregate counts plus "expiring soon" warnings right after redemption.

**Root cause (assumed, wrong):** the previous year's benefit cycle was lapsing while the new cycle's licenses activated, and the overlap window was what the doubled count and the expiry warnings both reflected — a transient state that would resolve itself once the old batch expired.

**What didn't work:** the theory didn't survive being checked against the actual expiry dates and order history. It also didn't hold up conceptually: if the old batch were simply due to lapse and get replaced 1:1, the *warning* would make sense, but it wouldn't explain why the aggregate count had already doubled *before* anything had actually expired — a straightforward handover shouldn't show both licenses' worth at once ahead of time.

**Fix:** cross-reference the aggregate and per-order views ([step 4](#4-cross-reference-the-two-views-side-by-side)).

### Issue 2 — Aggregate view makes stacking look like growth

**Observed:** SKU counts roughly doubled (35 → 70, 10 → 15) in Entra ID.

**Root cause:** each year's benefit redemption creates a brand-new license batch stacked on top of the previous year's. Entra's aggregate SKU view just adds the orders together with no visual distinction between them.

**Fix:** use the per-order breakdown from Partner Center / "Your Products" to see each redemption as its own order with its own dates.

### Issue 3 — No self-service way to merge stacked orders

**Observed:** stacked batches from different years can't be consolidated.

**Root cause:** a **structural pattern in how Microsoft's partner benefits program provisions each year's redemption** — not a one-time cleanup task. Each year's redemption just adds another layer, and it recurs — a little worse — every year. Left alone indefinitely, the aggregate license view will keep growing less representative of what's actually needed versus what's just historically accumulated, and every future "expiring soon" warning will need the same cross-referencing exercise to distinguish "an old, superseded batch naturally ending" from "an actual gap opening up."

**Fix:** none available self-service. Document the pattern so it's recognized immediately next time.

## Lessons Learned

- **A doubled license count right after a benefit/subscription renewal isn't automatically a duplication bug or a gap — check whether it's additive stacking from how the provisioning program itself works, before assuming either.**
- **When a tenant-wide summary view (aggregate SKU counts) and a program-specific view (per-order breakdown) tell different-shaped stories, cross-reference them directly rather than trusting either one in isolation.** The aggregate view here made stacking indistinguishable from real growth; the per-order view is what actually resolved it.
- **An "expiring soon" warning is only alarming if the thing expiring is actually load-bearing.** Once it was clear the new order was additive rather than a replacement, the old order's expiry stopped being a coverage risk and became just an artifact to expect.
- **If a platform's provisioning model creates a recurring, structural side effect (not a one-off bug), document the pattern itself rather than just resolving the immediate instance** — the same "alarming-looking but benign" signal will reappear on the same schedule next time, and a documented pattern turns a re-investigation into a five-minute check.

## Quick Reference Checklist

| If you see… | Do this |
|---|---|
| SKU counts roughly doubled right after benefits redemption | Compare Entra aggregate view with Partner Center per-order breakdown |
| "N subscriptions will expire soon" after redemption | Check which order is expiring — if it's the old order and the new one is active and additive, no coverage risk |
| New redemption shows as a separate order with its own dates | Expected stacking — no self-service merge; note it and move on |
| Same alarm next year | Annual pattern — run the cross-check, don't re-investigate from scratch |
