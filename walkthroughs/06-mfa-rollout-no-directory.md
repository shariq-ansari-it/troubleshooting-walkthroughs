# Rolling Out MFA to Servers With No Directory Service

**Stack:** Cisco Duo, RDP/RDS, local Windows accounts (no Active Directory)

## TL;DR

Most MFA rollout guidance assumes you have a directory service to sync from. These servers didn't — local Windows accounts only, no AD, no SSO, nothing centralized to point an MFA provider at. That single constraint reshapes almost every other decision: how users get provisioned, how the same person is matched across machines with inconsistent usernames, how offboarding works, and — critically — how you avoid locking yourself out of a production server while testing an authentication change on it.

## The constraint that drives everything else

Directory sync (the standard, low-maintenance way most MFA products expect to be provisioned) needs a directory to sync *from*. Local-account-only servers have no such thing — each server has its own independent set of Windows accounts with no central identity source. That rules out directory sync entirely, and downstream of that:

- **Provisioning is manual, or bulk-import from a CSV**, keyed on username as the only mandatory match field — there's no stable unique identifier to provision against otherwise.
- **The same person can have different local usernames on different servers** (e.g., `jsmith` on one box, `j.smith` on another) — depending on how those local accounts were originally created. An MFA provider that matches purely on typed username needs the *same person* mapped to multiple username aliases under one identity object, not treated as separate people or forced into a single renamed account across every server.
- **Access scoping and offboarding are fully manual, per server.** There's no security group to remove someone from that instantly revokes access everywhere — offboarding means disabling the local Windows account *and* the MFA-side user object, separately, on every server they had access to.

None of this is a reason not to do MFA — RDP/RDS exposed without it is a real risk — but it does mean the rollout plan has to be designed around these constraints explicitly rather than assuming a directory-based playbook will just work.

## Piloting on the right server first

Before touching anything customer-facing, the plan started with an internal-only pilot server — deliberately choosing New User Policy = "Allow Access" and Authentication Policy = "Bypass" for the first install, so the MFA agent could be validated as installed and functioning *before* actually enforcing anything. Flipping a brand-new, untested authentication layer straight to "enforce" on first install is how people lock themselves out of servers they can only reach via RDP.

## Break-glass planning, before installing anything

For a change that can lock out remote access if it goes wrong, the plan explicitly went through the failure mode *first*:

- Confirm out-of-band console access (hypervisor console, not RDP) works *right now*, before making any change — and keep a second console session open during testing, so there's a live way in if RDP access breaks.
- Have any disk-encryption recovery key on hand in case a botched agent install requires deeper recovery.
- Create a dedicated break-glass local admin account and set its MFA status to a permanent bypass (not just "hope console access is still there") — a second line of defense that doesn't depend on the hypervisor console being available at the exact moment it's needed.
- Restrict the MFA prompt to RDP logons only, not console logons — so a hypervisor console session bypasses the MFA layer entirely as a standing, permanent recovery path, not just during the pilot.

## Two smaller decisions worth calling out

- **Fail-open vs. fail-closed**, for what happens if the MFA provider's cloud service is briefly unreachable during a logon attempt: fail-open (allow access rather than lock everyone out) was the deliberate choice for the pilot phase, prioritizing availability over strict enforcement while the rollout is still being validated — expected to be revisited once the rollout is proven stable.
- **Agent version pinning**: checked the vendor's own release notes before picking an installer version, since a specific *earlier* patch version had a known bug where a silent/unattended upgrade could strip out the registry keys the agent needs to function — install directly on the fixed version rather than "latest" blindly, and verify the installer's checksum against the vendor's published hashes before running it on a production server.

## Scope discipline

Extending this same MFA layer to customer-facing servers (where actual customers, not staff, would need to enrol) was treated as an explicitly separate decision — different stakeholders, different support burden (a customer locked out by MFA needs a support path that doesn't exist yet for internal staff), and likely a separate policy configuration so customer authentication rules can be tuned independently from staff rules. Bundling "prove this works internally" and "roll it out to customers" into one project would have meant either moving too fast on the customer-facing piece or blocking the internal pilot on decisions that don't need to be made yet.

## Takeaways

- **"No directory service" isn't a blocker for MFA, but it does mean every provisioning/offboarding assumption from directory-based rollouts needs to be re-derived manually** — plan for username-alias mapping and per-server manual offboarding from day one rather than discovering the gap mid-rollout.
- **Pilot with the authentication policy set to bypass/allow, not enforce, until the agent itself is confirmed working.** Validate the plumbing before turning on the thing that can lock you out.
- **For any change that can break remote access to a server, write out the break-glass path before making the change, not after something goes wrong.** Out-of-band console access, a bypass-flagged break-glass account, and a policy scope that always exempts console logons cost little to set up and are exactly the thing you'll need at 6pm on a Friday if something misbehaves.
- **Read the vendor's release notes for known upgrade bugs before picking an installer version** — "latest" isn't automatically safest if a recent patch has a documented regression.
