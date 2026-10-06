# Rolling Out MFA to Servers With No Directory Service

## Purpose of this Document

A case study and planning reference for rolling out MFA on RDP/RDS servers that use local Windows accounts only — no Active Directory, no SSO, nothing centralized to point an MFA provider at. That single constraint reshapes provisioning, cross-server identity matching, offboarding and, critically, how to avoid locking yourself out of a production server while testing an authentication change on it.

It is intentionally written to:

- Give the rollout path that was planned, in order, so it can be repeated on the next server
- Keep that path separate from the constraints and traps that shaped it
- Record each constraint with its cause and the decision taken, so it can be recognised quickly next time
- Make the break-glass planning explicit, before anything is installed

## Environment

- Cisco Duo (MFA agent for Windows logon / RDP)
- Windows servers accessed over RDP/RDS
- Local Windows accounts only — no Active Directory, no directory sync, no SSO
- Hypervisor console as the out-of-band access path
- Internal-only pilot server first; customer-facing servers out of scope for this phase

---

## Correct End-to-End Process (Authoritative)

This is the clean path to follow for future rollouts on directory-less servers.

### 1. Plan Provisioning Without a Directory

Directory sync is off the table (see [Issue 1](#issue-1--no-directory-to-sync-from)). Plan for:

- **Manual provisioning, or bulk import from a CSV**, keyed on username as the only mandatory match field
- **Username aliases** — map the same person's different local usernames on different servers to one MFA identity object (see [Issue 2](#issue-2--same-person-different-local-usernames-on-different-servers))
- **Per-server manual offboarding** — disable the local Windows account *and* the MFA-side user object, separately, on every server the person had access to

### 2. Pick the Pilot Server

Start on an **internal-only pilot server** — nothing customer-facing.

### 3. Write the Break-Glass Path Before Installing Anything

For a change that can lock out remote access, go through the failure mode first:

- Confirm **out-of-band console access** (hypervisor console, not RDP) works *right now*, before making any change — and keep a **second console session open** during testing, so there's a live way in if RDP access breaks.
- Have any **disk-encryption recovery key** on hand in case a botched agent install requires deeper recovery.
- Create a **dedicated break-glass local admin account** and set its MFA status to a **permanent bypass** — a second line of defense that doesn't depend on the hypervisor console being available at the exact moment it's needed.
- **Restrict the MFA prompt to RDP logons only**, not console logons — so a hypervisor console session bypasses the MFA layer entirely as a standing, permanent recovery path, not just during the pilot.

### 4. Choose the Agent Version

- Read the vendor's release notes before picking an installer version (see [Issue 4](#issue-4--earlier-agent-patch-version-could-strip-its-own-registry-keys-on-upgrade)).
- Install directly on the fixed version rather than "latest" blindly.
- Verify the installer's checksum against the vendor's published hashes before running it on a production server.

### 5. Install in Non-Enforcing Mode

First install on the pilot server with:

- **New User Policy** = "Allow Access"
- **Authentication Policy** = "Bypass"

This validates the agent as installed and functioning *before* actually enforcing anything (see [Issue 3](#issue-3--enforcing-on-first-install-risks-lockout)).

Also set the offline behaviour — **fail-open** for the pilot phase (see [Issue 5](#issue-5--fail-open-vs-fail-closed-when-the-mfa-cloud-is-unreachable)).

### 6. Validate the Plumbing, Then Enforce

Only once the agent itself is confirmed working does enforcement get turned on.

### 7. Treat Customer-Facing Servers as a Separate Project

Extending MFA to customer-facing servers is an explicitly separate decision (see [Issue 6](#issue-6--pressure-to-bundle-the-customer-facing-rollout)).

**If this all happens → MFA agent validated on the internal pilot server before enforcement, with standing recovery paths in place.**

---

## Issues Encountered in This Case

### Issue 1 — No directory to sync from

**Observed:** most MFA rollout guidance assumes a directory service to sync from. These servers had local Windows accounts only — each server with its own independent set of accounts and no central identity source.

**Root cause:** directory sync (the standard, low-maintenance way most MFA products expect to be provisioned) needs a directory to sync *from*.

**Fix:** provisioning is manual, or bulk import from a CSV, keyed on username as the only mandatory match field — there's no stable unique identifier to provision against otherwise. Access scoping and offboarding are also fully manual, per server: there's no security group to remove someone from that instantly revokes access everywhere.

None of this is a reason not to do MFA — RDP/RDS exposed without it is a real risk — but the rollout plan has to be designed around these constraints explicitly rather than assuming a directory-based playbook will just work.

### Issue 2 — Same person, different local usernames on different servers

**Observed:** depending on how local accounts were originally created, one person can be e.g. `jsmith` on one box and `j.smith` on another.

**Root cause:** the MFA provider matches purely on typed username.

**Fix:** map the *same person* to multiple username aliases under one identity object — not treated as separate people, and not forced into a single renamed account across every server.

### Issue 3 — Enforcing on first install risks lockout

**Observed:** the servers can only be reached via RDP, and the MFA agent is a brand-new, untested authentication layer.

**Root cause:** flipping an untested authentication layer straight to "enforce" on first install is how people lock themselves out of servers they can only reach via RDP.

**Fix:** New User Policy = "Allow Access" and Authentication Policy = "Bypass" for the first install, plus the break-glass measures in [step 3](#3-write-the-break-glass-path-before-installing-anything).

### Issue 4 — Earlier agent patch version could strip its own registry keys on upgrade

**Observed:** the vendor's release notes documented that a specific *earlier* patch version had a known bug where a silent/unattended upgrade could strip out the registry keys the agent needs to function.

**Fix:** install directly on the fixed version rather than "latest" blindly, and verify the installer's checksum against the vendor's published hashes.

### Issue 5 — Fail-open vs fail-closed when the MFA cloud is unreachable

**Observed:** a decision is needed for what happens if the MFA provider's cloud service is briefly unreachable during a logon attempt.

**Fix:** fail-open (allow access rather than lock everyone out) was the deliberate choice for the pilot phase, prioritizing availability over strict enforcement while the rollout is still being validated — expected to be revisited once the rollout is proven stable.

### Issue 6 — Pressure to bundle the customer-facing rollout

**Observed:** the same MFA layer could be extended to customer-facing servers, where actual customers, not staff, would need to enrol.

**Root cause:** different stakeholders, different support burden (a customer locked out by MFA needs a support path that doesn't exist yet for internal staff), and likely a separate policy configuration so customer authentication rules can be tuned independently from staff rules.

**Fix:** treated as an explicitly separate decision. Bundling "prove this works internally" and "roll it out to customers" into one project would have meant either moving too fast on the customer-facing piece or blocking the internal pilot on decisions that don't need to be made yet.

## Lessons Learned

- **"No directory service" isn't a blocker for MFA, but every provisioning/offboarding assumption from directory-based rollouts needs to be re-derived manually** — plan for username-alias mapping and per-server manual offboarding from day one rather than discovering the gap mid-rollout.
- **Pilot with the authentication policy set to bypass/allow, not enforce, until the agent itself is confirmed working.** Validate the plumbing before turning on the thing that can lock you out.
- **For any change that can break remote access to a server, write out the break-glass path before making the change, not after something goes wrong.** Out-of-band console access, a bypass-flagged break-glass account, and a policy scope that always exempts console logons cost little to set up and are exactly the thing you'll need at 6pm on a Friday if something misbehaves.
- **Read the vendor's release notes for known upgrade bugs before picking an installer version** — "latest" isn't automatically safest if a recent patch has a documented regression.

## Quick Reference Checklist

| If you see… | Do this |
|---|---|
| No directory to sync users from | Manual provisioning or CSV bulk import, keyed on username |
| Same person with different usernames across servers | Map them as username aliases under one MFA identity object |
| Someone leaving | Disable the local Windows account *and* the MFA user object, on every server |
| About to install the agent on a server | Confirm console access now, keep a second console session open, have the disk-encryption recovery key ready |
| No guaranteed way in if MFA misbehaves | Break-glass local admin with permanent MFA bypass; MFA on RDP logons only, not console |
| First install | New User Policy = "Allow Access", Authentication Policy = "Bypass" |
| Choosing an installer version | Check release notes for known upgrade bugs; verify checksum against published hashes |
| MFA cloud unreachable during pilot | Fail-open; revisit once stable |
| Request to include customer-facing servers | Separate decision, separate policy configuration |
