# An Orphaned Subscription Blocked by Two Independently Deprecated Transfer Paths

## TL;DR

A legacy Azure subscription needed its administrative ownership transferred to a current team member, and every self-service path for doing that turned out to be blocked — for two completely unrelated reasons that happened to overlap on the same subscription. The classic "Change service administrator" control was permanently disabled because Microsoft had fully retired that entire administrative model. The fallback option, a billing-ownership transfer, was then rejected too — because the subscription's billing was CSP/Partner-managed, a category explicitly excluded from that self-service flow regardless of what role is held.

## Purpose of this Document

A case study and reference runbook for the problem summarised in the TL;DR above.

It is intentionally written to:

- Give the recommended resolution path, so time isn't spent hunting for a self-service fix that doesn't exist
- Keep that path separate from the two dead ends hit along the way
- Record each dead end with its symptom and cause, so the pattern is recognised quickly next time

## Environment

- Azure subscription administration — legacy subscription old enough to have used the classic Account Administrator / Service Administrator model
- Microsoft billing models — MOSP (Microsoft Online Subscription Program) self-service transfer, CSP (Cloud Solution Provider) / Partner billing
- Azure Portal — subscription **Properties** blade

---

## Correct End-to-End Process (Authoritative)

An older Azure subscription's administrative access needed tidying up — the classic "Account Administrator" / "Service Administrator" roles associated with it no longer matched who should actually control it. This is the path to follow when that happens.

### 1. Treat a Greyed-Out "Change administrator" as a Retired Feature

If the "Change administrator" control in the subscription's **Properties** blade is greyed out, check Microsoft's own deprecation timeline for the classic admin model before assuming a role or account issue. It is very likely a fully retired feature, not a permissions problem — see [Issue 1](#issue-1--change-administrator-control-permanently-greyed-out).

### 2. Check the Subscription's Billing Type Before Trying a Billing-Ownership Transfer

The documented fallback is a **billing ownership transfer** (the self-service MOSP flow). It has its own exclusions, independent of the classic-admin retirement: CSP/Partner-billed subscriptions aren't eligible. Check how the subscription is billed before assuming this will work — see [Issue 2](#issue-2--billing-ownership-transfer-rejected).

### 3. Stop Searching for a Self-Service Fix

If both paths are blocked, neither is a bug to fix or a permission to request — both are Microsoft's deliberate platform/program boundaries. The remaining paths are:

- **A Microsoft-support-brokered process** — similar in spirit to the billing-account-transfer support case in [walkthrough 01](01-billing-credit-wrong-account.md), or
- **Address it structurally** — provision a fresh subscription under current, correctly-modeled billing/administration and migrate workloads across, rather than continuing to try to "fix" administration on a subscription whose underlying program type no longer has a self-service ownership path at all.

**If this all happens → ownership is resolved through support or a fresh, correctly-modeled subscription.**

---

## Issues Encountered in This Case

### Issue 1 — "Change administrator" control permanently greyed out

**Observed:** the obvious fix — use the Azure Portal's subscription properties to change the service administrator to someone current — wasn't available. The "Change administrator" control in the subscription's Properties blade was permanently greyed out.

**Root cause:** not a permissions issue — a genuinely retired feature. Microsoft deprecated the classic Account Administrator/Service Administrator model in stages, with the underlying capability withdrawn first and the UI control itself removed entirely in a later platform update.

**What didn't work:** looking for the "right" role or the "right" account to be signed in as. The control no longer functions for any account, because the feature it drove has been fully retired.

**Fix:** none on this path — move to the billing-ownership fallback or the paths in [step 3](#3-stop-searching-for-a-self-service-fix).

### Issue 2 — Billing-ownership transfer rejected

**Observed:** Azure's documented fallback, a **billing ownership transfer** — moving the subscription to a different billing account/owner, a self-service MOSP flow available even without the classic admin model — failed with an explicit error to the effect of *"Transfer is not supported for your subscription type."*

**Root cause:** the subscription's billing was managed through a **CSP (Cloud Solution Provider) / Partner-billed** arrangement — a billing relationship Microsoft explicitly excludes from the standard self-service MOSP transfer flow, regardless of which role or account is attempting it. Not a permissions gap to work around; CSP-billed subscriptions are categorically out of scope for that self-service tool by design.

**Fix:** none self-service. Two independent dead ends, from two unrelated causes, converged on the same subscription:

1. The administrative-ownership path is gone because the feature itself was retired platform-wide.
2. The billing-ownership path is blocked because of how this specific subscription is billed (CSP/Partner), not because of any misconfiguration on it.

Use the paths in [step 3](#3-stop-searching-for-a-self-service-fix).

## Lessons Learned

- **A greyed-out "Change administrator" control on an old Azure subscription is very likely a fully retired feature, not a permissions problem** — worth checking Microsoft's own deprecation timeline before assuming a role or account issue.
- **The billing-ownership-transfer fallback has its own exclusions, independent of the classic-admin-model retirement** — CSP/Partner-billed subscriptions specifically aren't eligible for the standard self-service MOSP transfer flow. Check the subscription's billing type before assuming this fallback will work.
- **When two unrelated legacy/administrative dead ends converge on the same old resource, that's often a signal the resource itself has aged past what self-service tooling supports at all** — rather than continuing to search for a self-service fix, it can be faster to involve Microsoft support directly, or to treat "stand up a fresh, correctly-modeled subscription and migrate" as the actual solution.
- **Legacy Azure subscriptions (particularly ones old enough to have used the classic Account/Service Administrator model) are worth auditing proactively** for exactly this kind of trapped state, rather than discovering it only when an ownership change is actually needed.

## Quick Reference Checklist

| If you see… | Do this |
|---|---|
| "Change administrator" greyed out in subscription Properties | Classic admin model is retired — check Microsoft's deprecation timeline, don't hunt for a role |
| *"Transfer is not supported for your subscription type."* | Check billing type — CSP/Partner-billed subscriptions are excluded from self-service MOSP transfer |
| Both paths blocked on the same old subscription | Microsoft support case, or fresh subscription under current billing + migrate workloads |
| Old subscription that used classic Account/Service Administrator | Audit it proactively before an ownership change is needed |
