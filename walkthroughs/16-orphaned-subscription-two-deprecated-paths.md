# An Orphaned Subscription Blocked by Two Independently Deprecated Transfer Paths

**Stack:** Azure subscription administration, Microsoft billing/CSP models

## TL;DR

A legacy Azure subscription needed its administrative ownership transferred to a current team member, and every self-service path for doing that turned out to be blocked — for two completely unrelated reasons that happened to overlap on the same subscription. The classic "Change service administrator" control was permanently disabled because Microsoft had fully retired that entire administrative model. The fallback option, a billing-ownership transfer, was then rejected too — because the subscription's billing was CSP/Partner-managed, a category explicitly excluded from that self-service flow regardless of what role is held.

## The situation

An older Azure subscription's administrative access needed tidying up — the classic "Account Administrator" / "Service Administrator" roles associated with it no longer matched who should actually control it. The obvious fix looked simple: use the Azure Portal's subscription properties to change the service administrator to someone current.

## Dead end 1: the classic admin model itself is gone

The "Change administrator" control in the subscription's Properties blade was permanently greyed out — not a permissions issue, a genuinely retired feature. Microsoft deprecated the classic Account Administrator/Service Administrator model in stages, with the underlying capability withdrawn first and the UI control itself removed entirely in a later platform update. This isn't fixable by finding the "right" role or the "right" account to be signed in as — the control simply no longer functions for any account, because the feature it drove has been fully retired.

## Dead end 2: the billing-ownership fallback, blocked for an unrelated reason

Azure's documented fallback for this kind of situation is a **billing ownership transfer** — moving the subscription to a different billing account/owner, which is a self-service (MOSP — Microsoft Online Subscription Program) flow available even without the classic admin model.

Attempting it failed with an explicit error to the effect of *"Transfer is not supported for your subscription type."* Investigating why turned up the actual cause: the subscription's billing was managed through a **CSP (Cloud Solution Provider) / Partner-billed** arrangement — a billing relationship Microsoft explicitly excludes from the standard self-service MOSP transfer flow, regardless of which role or account is attempting it. This isn't a permissions gap to work around; CSP-billed subscriptions are categorically out of scope for that particular self-service tool by design.

## Where that leaves things

Two independent dead ends, from two unrelated causes, both converging on the same subscription:

1. The administrative-ownership path is gone because the feature itself was retired platform-wide.
2. The billing-ownership path is blocked because of how this specific subscription is billed (CSP/Partner), not because of any misconfiguration on it.

Neither is a bug to fix or a permission to request — both are Microsoft's deliberate platform/program boundaries. The actual remaining paths are either a Microsoft-support-brokered process (similar in spirit to a billing-account-transfer support case), or addressing it structurally — e.g., provisioning a fresh subscription under current, correctly-modeled billing/administration and migrating workloads across, rather than continuing to try to "fix" administration on a subscription whose underlying program type no longer has a self-service ownership path at all.

## Takeaways

- **A greyed-out "Change administrator" control on an old Azure subscription is very likely a fully retired feature, not a permissions problem** — worth checking Microsoft's own deprecation timeline before assuming a role or account issue.
- **The billing-ownership-transfer fallback has its own exclusions, independent of the classic-admin-model retirement** — CSP/Partner-billed subscriptions specifically aren't eligible for the standard self-service MOSP transfer flow. Check the subscription's billing type before assuming this fallback will work.
- **When two unrelated legacy/administrative dead ends converge on the same old resource, that's often a signal the resource itself has aged past what self-service tooling supports at all** — rather than continuing to search for a self-service fix, it can be faster to involve Microsoft support directly, or to treat "stand up a fresh, correctly-modeled subscription and migrate" as the actual solution.
- **Legacy Azure subscriptions (particularly ones old enough to have used the classic Account/Service Administrator model) are worth auditing proactively** for exactly this kind of trapped state, rather than discovering it only when an ownership change is actually needed.
