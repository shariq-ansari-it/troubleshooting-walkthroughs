# The License Count That Doubled Overnight — Stacking, Not a Gap

**Stack:** Microsoft 365 licensing, Microsoft Partner Center, Microsoft Entra ID

## TL;DR

Right after redeeming an annual Microsoft partner benefits package, the tenant's aggregate license counts for two SKUs abruptly doubled — one went from 35 to 70, another from 10 to 15 — alongside a batch of "N subscriptions will expire soon" warnings. The obvious read was either a licensing gap about to open up, or an overlap that needed cleaning up. Neither was right. Cross-referencing two different Microsoft views of the same tenant against each other showed the real mechanism: each year's benefit redemption creates a brand-new license batch stacked on top of the previous year's, rather than renewing or merging it. Nothing was broken, but there's no self-service way to clean it up, and it recurs — a little worse — every year.

## The symptom

Two things landed close together, right after that year's Partner Success benefits redemption:
- Entra ID's aggregate license view showed SKU counts that had roughly doubled versus before the redemption.
- The admin center was sending "N subscriptions will expire soon" notifications for what looked like a meaningful chunk of licenses.

Read together, those two signals suggest a specific, alarming story: a batch of licenses is about to lapse, and the doubled count is masking an actual coverage gap that's about to become visible the moment the expiring batch drops off.

## First theory — and why it was wrong

The first, reasonable-sounding explanation: the previous year's benefit cycle was lapsing while the new cycle's licenses activated, and the overlap window was what the doubled count and the expiry warnings both reflected — a transient state that would resolve itself once the old batch actually expired.

That theory didn't survive being checked against the actual expiry dates and order history. It also didn't hold up conceptually: if the old batch were simply due to lapse and get replaced 1:1, the *warning* would make sense, but it wouldn't explain why the aggregate count had already doubled *before* anything had actually expired — a straightforward handover shouldn't show both licenses' worth at once ahead of time the way this did.

## Getting to the real cause

The fix for figuring out what was actually going on was cross-referencing two views of the same tenant that don't usually get looked at side by side:

- **Entra ID's aggregate license/SKU view** — a single rolled-up count per SKU, with no indication of how many separate underlying orders make up that count.
- **The per-order breakdown from Partner Center / the "Your Products" summary** — which lists each benefit redemption as its own distinct order, each with its own start and end date.

Lined up against each other, the picture was clear: **the current year's redemption had created an entirely new, separate license order, additive to the previous year's order, rather than renewing or replacing it.** The aggregate SKU view in Entra just adds these orders together with no visual distinction between them — so from that view alone, a stacked total looks identical to "a real increase in purchased licenses." The expiry warnings were genuine — but they were warning about the *old* order's natural end date, which was never a threat to actual coverage, since the new order was already active and additive, not a replacement waiting to take over.

## Why this matters going forward

This isn't a one-time cleanup task — it's a **structural pattern in how Microsoft's partner benefits program provisions each year's redemption.** There's no self-service tool to merge stacked batches from different years into one clean order; each year's redemption just adds another layer. Left alone indefinitely, the aggregate license view will keep growing less representative of what's actually needed versus what's just historically accumulated, and every future "expiring soon" warning will need the same cross-referencing exercise to distinguish "an old, superseded batch naturally ending" from "an actual gap opening up."

Rather than trying to force a one-time fix, the resolution was to document the pattern itself as an annual, expected event — so next year's identical-looking alarm gets recognized immediately instead of re-investigated from scratch.

## Takeaways

- **A doubled license count right after a benefit/subscription renewal isn't automatically a duplication bug or a gap — check whether it's additive stacking from how the provisioning program itself works, before assuming either.**
- **When a tenant-wide summary view (aggregate SKU counts) and a program-specific view (per-order breakdown) tell different-shaped stories, cross-reference them directly rather than trusting either one in isolation.** The aggregate view here made stacking indistinguishable from real growth; the per-order view is what actually resolved it.
- **An "expiring soon" warning is only alarming if the thing expiring is actually load-bearing.** Once it was clear the new order was additive rather than a replacement, the old order's expiry stopped being a coverage risk and became just an artifact to expect.
- **If a platform's provisioning model creates a recurring, structural side effect (not a one-off bug), document the pattern itself rather than just resolving the immediate instance** — the same "alarming-looking but benign" signal will reappear on the same schedule next time, and a documented pattern turns a re-investigation into a five-minute check.
