# Guest Account Sprawl From "Helpful" Sharing Links

**Stack:** Microsoft Entra ID (Azure AD), SharePoint/OneDrive, Microsoft Graph audit logs, PnP PowerShell

## TL;DR

A support team was sharing internal training recordings with clients using "Specific people" links in OneDrive — the option that felt most secure. Each one of those links silently created a *permanent* external guest account in the directory, whether or not the recipient ever needed ongoing access. Traced it via directory audit logs, then made a deliberate tradeoff: switch the sharing default to anonymous, time-boxed links for this specific low-sensitivity content, instead of reflexively granting more people the ability to invite guests.

## How it started

Two separate support requests, weeks apart, both boiling down to: "I tried to share a recording/file with an external client and got an error saying guest invitations aren't allowed." The fix each time looked simple — grant the person the built-in **Guest Inviter** directory role, which does exactly one thing: lets its holder send B2B guest invites, independent of whatever the tenant-wide "members can invite guests" setting is. No other permissions come with it.

After the second occurrence, the role was granted proactively to a few more people expected to hit the same wall. That's where it stopped being simple.

## What actually happened

Every time someone used a "Specific people" share link — the option that sounds the most restrictive and secure — Entra silently created a permanent guest account for that external recipient, if one didn't already exist. This is expected, documented behavior: identity-verified sharing needs an identity to verify against, so an invite goes out and a guest object gets created regardless of whether the recipient ever accepts it or needs continued access.

Multiple people now holding Guest Inviter, all sharing training material with a rotating set of external contacts, meant a steady accumulation of guest accounts — a chunk of which were invited but never signed in at all (recipient never needed to actually visit anything, or just watched an emailed preview).

## Tracing it

Confirmed the pattern via Microsoft Graph's directory audit logs, filtering for `activityDisplayName eq 'Invite external user'`. The initiating application on every relevant entry was SharePoint Online itself — not a person directly running an "invite guest" flow, but the sharing feature doing it as a side effect. That confirmed the mechanism: this wasn't people misusing the Guest Inviter role, it was the *sharing link type itself* that caused it.

Checked why anonymous "Anyone with the link" sharing wasn't already an option — the tenant's OneDrive sharing capability was capped at "existing guests" (identity-verified sharing only), so anonymous links weren't even offered as a choice in the share dialog. That's what pushed people toward identity-verified guest invites for content that didn't actually need per-person revocable access.

## The fix, and the tradeoff behind it

Rather than keep granting Guest Inviter to more people (which only makes the underlying sprawl mechanism run faster), the actual lever was the sharing policy itself:

```powershell
Set-PnPTenant `
  -SharingCapability ExternalUserAndGuestSharing `
  -OneDriveSharingCapability ExternalUserAndGuestSharing `
  -DefaultSharingLinkType AnonymousAccess `
  -RequireAnonymousLinksExpireInDays 60
```

This raises OneDrive's sharing ceiling to allow anonymous, time-boxed links — no guest account gets created per share, and links auto-expire after 60 days.

Two things worth calling out about this change:
- **It only affects OneDrive (and any newly created SharePoint site going forward).** Raising the tenant-wide ceiling did **not** retroactively loosen existing SharePoint site collections — each one already had its own, more restrictive sharing capability locked in at creation time. Verified this with `Get-PnPTenantSite` before assuming the change was universal.
- **Anonymous links are a real, deliberate tradeoff, not a free upgrade.** A forwardable link with no per-person revocation and no way to claw back an already-downloaded copy is a worse security posture *in general* than identity-verified access. It was the right call **specifically** because the content in question was low-value outside the context of an existing paying customer relationship (product training material) — not a decision to generalize to every sharing scenario in the tenant.

Cleanup: removed the Guest Inviter role from everyone it had been granted to for this purpose, and deleted the guest accounts that had been invited but never signed in. Left alone the handful of guest accounts that were genuinely, actively being used for ongoing external collaboration.

## Takeaways

- **"Specific people" sharing isn't free of side effects just because it sounds more restrictive than a public link — it can create standing identity objects you now have to manage.**
- **Directory audit logs (`Invite external user`, filtered by initiating app) are the fast way to tell whether guest sprawl is coming from a human process or a platform feature doing it automatically.**
- **Don't default to widening a permission (like Guest Inviter) to solve a symptom — check whether the underlying sharing mechanism is the actual lever first.**
- **A tenant-wide sharing policy change doesn't automatically cascade to resources that already have their own explicit, stricter setting.** Verify before assuming a policy change reaches everywhere it logically "should."
- **Anonymous vs. identity-verified sharing is a security tradeoff to make deliberately per use case, not a blanket policy** — match it to how sensitive the actual content is, not just to which team is asking.
