# The Sponsorship Credit That Wouldn't Apply

**Stack:** Azure, Cost Management API, Microsoft billing accounts/profiles, Azure Portal support tickets

## TL;DR

An Azure subscription kept getting billed in full (~£300+/month) despite a $5,000 credit having been redeemed for it. The credit was real and showing as available — just not against this subscription. Root cause: the credit had landed on one billing account, while the subscription itself was invoiced under a completely different, older billing account. No self-service tool covers moving a subscription between billing accounts in that direction, and the "obvious" CLI path to open a support ticket was blocked by an unrelated support-plan restriction. Fixed via a Microsoft-support-brokered subscription transfer that required an approval email from each billing account's owner.

## The symptom

A subscription originally issued under a cloud sponsorship program had burned through its original credit and been auto-converted to standard pay-as-you-go billing — expected behavior once a sponsorship credit runs out. After a subsequent benefits renewal, a new credit was redeemed to cover it going forward. The redemption succeeded and the credit showed as available in the billing portal. The subscription kept getting charged in full anyway.

## Investigation

First step was confirming the credit actually existed and wasn't just a display glitch. Querying the Cost Management API for the billing account showed the credit correctly: full balance available, zero consumed. So the money was there — it just wasn't being drawn against this subscription's usage.

Next was checking exactly which billing account and billing profile the subscription itself was invoiced under, via the subscription's own Properties blade in the Portal. That's where the mismatch showed up: the subscription was billed under a different billing account entirely — an older, legacy-style account — not the one the new credit had been redeemed into.

Two Microsoft billing accounts, one subscription attached to the wrong one relative to where the money was sitting. Credits don't reach across billing accounts. Structurally, there was no way for this subscription to draw on that balance while attached to the other account.

## Dead ends

**"Switch Offer" self-service.** Azure subscriptions have a self-service flow for converting between commercial offer types (e.g., pay-as-you-go to a modern Microsoft Customer Agreement / "Azure Plan" offer). Tried it — the tool reported no eligible conversions available. Direction matters here: the tooling covers some offer transitions but not this one.

**CLI/API support ticket creation.** The natural next step for a billing issue is opening a support case. Azure CLI has a command for exactly that (`az support tickets create`). It failed immediately with `InvalidSupportPlan` — the API requires a paid support plan to create a ticket via that path, *regardless of ticket category*, even for a billing-only question that costs nothing to raise. The Azure Portal's own Help + Support blade doesn't have this restriction — it lets free/basic-tier accounts raise billing tickets with no paid plan required. The CLI and the Portal enforce different rules for what should be the same operation. Lesson: for a subscription on a free/basic support plan, don't bother with the CLI ticket path — go straight to the Portal.

## Resolution path

Raised a ticket through the Portal (Subscription Management → Benefits/Offers → Activation or extension request category), explaining the mismatch and asking what the correct fix was.

The support engineer's proposal wasn't a billing-ownership change — it was a full **subscription transfer** to the other billing account. That operation requires an approval email from the *owner of each side*: the current (source) billing account owner and the destination billing account owner, each independently confirming the transfer in writing back on the case thread. Since the destination account's owner happened to also be the requester in this case, only one external approval was actually needed — but tracking down who legitimately owned the source (older/legacy) billing account, since there was no visible role assignment on it from this side, took some digging.

Once both approvals were in, Microsoft executed the transfer. The subscription moved to the modern billing account, and — notably — reverted its display name/offer type back to reflect the sponsorship program, now correctly drawing against the existing credit balance.

## Takeaways

- **A redeemed credit and a subscription's actual billing account are two separate facts** — always check both independently when a "the credit isn't applying" symptom shows up. The Portal's subscription Properties blade and a Cost Management API query for the billing account are the two places to look.
- **Self-service offer-switching tools are directional** — a "no conversions available" result doesn't mean nothing can be done, it means this specific tool doesn't cover this specific direction.
- **The Azure CLI and the Azure Portal don't always enforce the same rules for the same nominal operation.** If a CLI command fails with a support-plan or entitlement error, try the Portal equivalent before assuming the whole operation is blocked.
- **Cross-billing-account subscription transfers are a two-sided approval process.** If you're not the owner of both billing accounts involved, budget time to identify and coordinate with whoever holds the other side.
