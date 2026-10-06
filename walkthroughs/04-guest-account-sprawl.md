# Guest Account Sprawl From "Helpful" Sharing Links

## Purpose of this Document

A case study of guest account sprawl caused by support staff sharing training recordings with clients via OneDrive "Specific people" links — the option that felt most secure, but which silently creates a *permanent* external guest account per recipient. Traced via directory audit logs, then fixed with a deliberate tradeoff: anonymous, time-boxed links for this low-sensitivity content instead of granting more people the ability to invite guests.

It is intentionally written to:

- Give the final sharing-policy change and cleanup that resolved it
- Separate it from the first fix (granting Guest Inviter) that worked per-ticket but fed the sprawl
- Help recognise both the "guest invitations aren't allowed" error and the sprawl pattern next time

## Environment

- Microsoft Entra ID (Azure AD) — B2B guests, `authorizationPolicy.allowInvitesFrom`, Guest Inviter directory role
- SharePoint Online / OneDrive sharing
- Microsoft Graph — directory audit logs, role assignments
- PnP PowerShell (`Set-PnPTenant`, `Get-PnPTenantSite`)

---

## Correct End-to-End Process (Authoritative)

### 1. Confirm the Cause of the "Guest Invitations Aren't Allowed" Error

Check via Microsoft Graph rather than guessing:

- tenant `authorizationPolicy.allowInvitesFrom` — here `adminsAndGuestInviters`
- the affected person's directory role assignments — here **zero**

Most staff have zero role assignments by default, so this is "most people can't invite guests unless someone deliberately grants it" — working as intended, not a misconfiguration. On a repeat ticket, the role-assignment check alone confirms it ([Issue 1](#issue-1--guest-invitations-arent-allowed-error)).

### 2. Check Whether Sharing Links Are Creating Guests

Query Microsoft Graph directory audit logs filtered on `activityDisplayName eq 'Invite external user'` and look at the initiating application.

Here every relevant entry was initiated by **SharePoint Online** — the sharing feature creating guests as a side effect, not a person running an invite flow. See [Issue 2](#issue-2--specific-people-links-create-permanent-guest-accounts).

### 3. Check the OneDrive Sharing Ceiling

Find out why anonymous "Anyone with the link" wasn't offered. Here the tenant's OneDrive sharing capability was capped at "existing guests" (identity-verified only), so anonymous links weren't a choice in the share dialog — pushing people to guest invites for content that didn't need per-person revocable access.

### 4. Change the Sharing Policy

Rather than keep granting Guest Inviter (which only makes the sprawl run faster), change the sharing policy itself:

```powershell
Set-PnPTenant `
  -SharingCapability ExternalUserAndGuestSharing `
  -OneDriveSharingCapability ExternalUserAndGuestSharing `
  -DefaultSharingLinkType AnonymousAccess `
  -RequireAnonymousLinksExpireInDays 60
```

This raises OneDrive's ceiling to allow anonymous, time-boxed links — no guest account per share, and links auto-expire after 60 days.

This is a deliberate tradeoff, not a free upgrade — see [Issue 3](#issue-3--anonymous-links-are-a-weaker-posture).

### 5. Verify Which Sites the Change Actually Reached

Run `Get-PnPTenantSite` before assuming the change is universal. It only affects OneDrive and newly created SharePoint sites — see [Issue 4](#issue-4--tenant-change-doesnt-reach-existing-sharepoint-sites).

### 6. Clean Up

- Remove the Guest Inviter role from everyone it was granted to for this purpose
- Delete guest accounts that were invited but never signed in
- Leave the guest accounts genuinely in active use for ongoing external collaboration

**If this all happens → clients get time-boxed links, and no new guest accounts accumulate from routine sharing.**

---

## Issues Encountered in This Case

### Issue 1 — "Guest invitations aren't allowed" error

**Observed:** two separate support requests, weeks apart, for different people: sharing a recording/file with an external client failed with an error saying guest invitations aren't allowed.

**Root cause:** `allowInvitesFrom` = `adminsAndGuestInviters`, and the person held no directory roles at all. Working as intended.

**What didn't work (long-term):** granting the built-in **Guest Inviter** role. It does exactly one thing — lets its holder send B2B guest invites, independent of the tenant-wide "members can invite guests" setting, with no other permissions. It fixed each ticket, and after the second occurrence was granted proactively to a few more people expected to hit the same wall. That's where it stopped being simple: more inviters meant faster sprawl ([Issue 2](#issue-2--specific-people-links-create-permanent-guest-accounts)).

**Fix:** the sharing-policy change in [step 4](#4-change-the-sharing-policy); Guest Inviter grants removed in [step 6](#6-clean-up).

The second occurrence was recognised quickly as the same root cause — a check of the person's role assignments (empty, as before) replaced re-deriving the whole `allowInvitesFrom` chain, turning a second full investigation into a thirty-second check.

### Issue 2 — "Specific people" links create permanent guest accounts

**Observed:** steady accumulation of guest accounts as several Guest Inviter holders shared training material with a rotating set of external contacts; a chunk were invited but never signed in (recipient never needed to visit anything, or just watched an emailed preview).

**Root cause:** expected, documented behaviour — identity-verified sharing needs an identity to verify against, so every "Specific people" link sends an invite and creates a guest object (if one doesn't exist), whether or not the recipient accepts or needs continued access. Audit logs showed SharePoint Online as the initiating app, so it was the *link type itself*, not misuse of the Guest Inviter role.

**Fix:** anonymous, expiring links for this content ([step 4](#4-change-the-sharing-policy)).

### Issue 3 — Anonymous links are a weaker posture

**Observed:** n/a — a design consideration.

**Root cause:** a forwardable link with no per-person revocation and no way to claw back an already-downloaded copy is a worse security posture *in general* than identity-verified access.

**Fix:** accepted **specifically** because the content (product training material) was low-value outside an existing paying customer relationship — not a decision to generalise to every sharing scenario in the tenant.

### Issue 4 — Tenant change doesn't reach existing SharePoint sites

**Observed:** raising the tenant-wide ceiling did **not** loosen existing SharePoint site collections.

**Root cause:** each existing site already had its own, more restrictive sharing capability locked in at creation time.

**Fix:** none needed for this case — verified with `Get-PnPTenantSite` rather than assuming the change reached everywhere.

## Lessons Learned

- **"Specific people" sharing isn't free of side effects just because it sounds more restrictive than a public link — it can create standing identity objects you now have to manage.**
- **Directory audit logs (`Invite external user`, filtered by initiating app) are the fast way to tell whether guest sprawl is coming from a human process or a platform feature doing it automatically.**
- **Don't default to widening a permission (like Guest Inviter) to solve a symptom — check whether the underlying sharing mechanism is the actual lever first.**
- **A tenant-wide sharing policy change doesn't automatically cascade to resources that already have their own explicit, stricter setting.** Verify before assuming a policy change reaches everywhere it logically "should."
- **Anonymous vs. identity-verified sharing is a security tradeoff to make deliberately per use case, not a blanket policy** — match it to how sensitive the actual content is, not just to which team is asking.
- **When someone hits a "not allowed" error for an action most people can't do by default, check their specific role assignments before assuming a tenant-wide policy is misconfigured.** Here, `allowInvitesFrom` restricting invites to admins and Guest Inviters was working exactly as intended — the "fix" was identifying who legitimately needed the role, not treating the restriction itself as a bug.
- **The same fix recurring for a different person, weeks later, is worth explicitly recognising as a pattern rather than re-investigating from first principles** — and worth asking, at that point, whether the *fix* itself (granting a role) is the right long-term answer or just deferring the same underlying tension (here, it turned out to be the latter — see above).

## Quick Reference Checklist

| If you see… | Do this |
|---|---|
| "Guest invitations aren't allowed" when sharing externally | Check `allowInvitesFrom` and the user's role assignments — likely working as intended |
| Same error again for a different person | Check their role assignments only — same root cause |
| Growing number of guest accounts, many never signed in | Audit logs: `Invite external user`, check initiating app |
| Initiating app is SharePoint Online | Sharing links are creating guests — fix the sharing policy, not inviter permissions |
| No "Anyone with the link" option in share dialog | OneDrive sharing capability capped at existing guests |
| Tenant sharing change made | `Get-PnPTenantSite` — existing sites keep their own stricter setting |
