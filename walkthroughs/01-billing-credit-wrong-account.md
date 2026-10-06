# The Sponsorship Credit That Wouldn't Apply

## Purpose of this Document

A case study of an Azure subscription that kept being billed in full (~£300+/month) despite a $5,000 credit having been redeemed for it — because the credit sat on one billing account while the subscription was invoiced under a different, older one. No self-service tool covered the move, and the CLI support-ticket path was blocked, so the fix was a Microsoft-support-brokered subscription transfer.

It is intentionally written to:

- Give the resolution path that actually worked, so it can be repeated without re-investigating
- Keep that path separate from the dead ends (self-service offer switching, CLI ticket creation)
- Help recognise the "credit available but not applying" pattern quickly next time

## Environment

- Azure subscription originally issued under a cloud sponsorship program, auto-converted to pay-as-you-go once its original credit ran out
- Two Microsoft billing accounts / billing profiles — one modern, one older legacy-style
- Cost Management API
- Azure CLI (`az support tickets create`)
- Azure Portal — subscription Properties blade, Help + Support

---

## Correct End-to-End Process (Authoritative)

### 1. Confirm the Credit Actually Exists

Rule out a display glitch first. Query the Cost Management API for the billing account the credit was redeemed into.

Expected: the credit shows correctly — full balance available, zero consumed. If so, the money is there; it just isn't being drawn against this subscription's usage.

### 2. Check Which Billing Account the Subscription Is Invoiced Under

**Azure Portal → the subscription → Properties** — note the billing account and billing profile.

Compare with the account from step 1. In this case the subscription was billed under a different billing account entirely — an older, legacy-style account — not the one the new credit had been redeemed into.

Credits don't reach across billing accounts. While the subscription is attached to the other account, there is structurally no way for it to draw on that balance.

### 3. Raise a Support Ticket Through the Portal

Don't go via the CLI on a free/basic support plan — see [Issue 2](#issue-2--az-support-tickets-create-fails-with-invalidsupportplan). (Self-service offer switching won't cover this direction either — see [Issue 1](#issue-1--switch-offer-reports-no-eligible-conversions).)

**Azure Portal → Help + Support** → category **Subscription Management → Benefits/Offers → Activation or extension request**. Explain the mismatch and ask what the correct fix is.

### 4. Agree a Subscription Transfer

The support engineer's proposal wasn't a billing-ownership change — it was a full **subscription transfer** to the other billing account.

### 5. Get Approval From Each Billing Account Owner

The transfer requires an approval email from the *owner of each side*, each independently confirming it in writing back on the case thread:

- the current (source) billing account owner
- the destination billing account owner

If the destination owner is also the requester, only one external approval is needed. Identifying who legitimately owns the source (older/legacy) billing account can take digging — see [Issue 3](#issue-3--source-billing-account-owner-not-visible).

### 6. Microsoft Executes the Transfer

Once both approvals are in, Microsoft executes the transfer. The subscription moves to the modern billing account and reverts its display name/offer type to reflect the sponsorship program.

**If this all happens → the subscription draws against the existing credit balance.**

---

## Issues Encountered in This Case

### Issue 1 — "Switch Offer" reports no eligible conversions

**Observed:** the subscription's self-service flow for converting between commercial offer types (e.g. pay-as-you-go to a modern Microsoft Customer Agreement / "Azure Plan" offer) reported no eligible conversions available.

**Root cause:** the tooling covers some offer transitions but not this one — direction matters.

**Fix:** none via self-service; "no conversions available" means this tool doesn't cover this direction, not that nothing can be done. Go to support ([step 3](#3-raise-a-support-ticket-through-the-portal)).

### Issue 2 — `az support tickets create` fails with `InvalidSupportPlan`

**Observed:** `az support tickets create` failed immediately with `InvalidSupportPlan`.

**Root cause:** the API requires a paid support plan to create a ticket via that path, *regardless of ticket category* — even for a billing-only question that costs nothing to raise. The Portal's Help + Support blade doesn't have this restriction; it lets free/basic-tier accounts raise billing tickets with no paid plan. The CLI and the Portal enforce different rules for what should be the same operation.

**Fix:** for a subscription on a free/basic support plan, skip the CLI ticket path and go straight to the Portal.

### Issue 3 — Source billing account owner not visible

**Observed:** no visible role assignment on the source (older/legacy) billing account from this side.

**Root cause:** the transfer needs written approval from that account's owner, and ownership wasn't discoverable from the requester's side.

**Fix:** track down the legitimate owner before (or while) the case runs — budget time for it.

## Lessons Learned

- **A redeemed credit and a subscription's actual billing account are two separate facts** — always check both independently when a "the credit isn't applying" symptom shows up. The Portal's subscription Properties blade and a Cost Management API query for the billing account are the two places to look.
- **Self-service offer-switching tools are directional** — a "no conversions available" result doesn't mean nothing can be done, it means this specific tool doesn't cover this specific direction.
- **The Azure CLI and the Azure Portal don't always enforce the same rules for the same nominal operation.** If a CLI command fails with a support-plan or entitlement error, try the Portal equivalent before assuming the whole operation is blocked.
- **Cross-billing-account subscription transfers are a two-sided approval process.** If you're not the owner of both billing accounts involved, budget time to identify and coordinate with whoever holds the other side.

## Quick Reference Checklist

| If you see… | Do this |
|---|---|
| Credit available in billing portal but subscription still billed in full | Compare the credit's billing account (Cost Management API) with the subscription's (Properties blade) |
| Credit balance full, zero consumed | Credit is real — it's on a different billing account from the subscription |
| "Switch Offer" shows no eligible conversions | Tool doesn't cover this direction — raise a support ticket |
| `InvalidSupportPlan` from `az support tickets create` | Raise the ticket in the Portal (Help + Support) instead |
| Support proposes a subscription transfer | Get written approval from both source and destination billing account owners on the case thread |
