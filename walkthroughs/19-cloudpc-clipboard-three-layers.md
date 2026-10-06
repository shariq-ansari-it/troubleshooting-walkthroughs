# The Cloud PC Clipboard That Needed Three Separate Fixes

## TL;DR

Copy-paste didn't work at all on a brand-new Cloud PC — not from the local device into the Cloud PC, not the other way, and not into a nested VM running inside it either. The obvious fix (find the "allow clipboard redirection" policy and turn it on) got applied, reported success, and changed nothing. It took three separate, independently-gated layers to actually fix it: a master on/off switch, a completely separate direction-and-format restriction sitting on top of it, and — once both of those were identified — a self-inflicted Intune policy conflict that silently blocked the fix from ever reaching the device for hours after it was "configured correctly."

## Purpose of this Document

A case study and reference runbook for the problem summarised in the TL;DR above.

It is intentionally written to:

- Give the working path — every gate that has to be opened, in one policy
- Keep it separate from the obvious explanations that were ruled out first
- Record each layer with its symptom and cause, so the pattern is recognized quickly on the next new Cloud PC

## Environment

- Windows 365 Cloud PC (newly provisioned)
- Microsoft Intune — Settings Catalog configuration profiles
- Azure Virtual Desktop (remote desktop client used to connect)
- Nested Hyper-V VM inside the Cloud PC, used for testing

---

## Correct End-to-End Process (Authoritative)

### 1. Start From the Platform Default

Cloud PCs disable clipboard, drive, and printer redirection by default on first provisioning. This is documented, intentional, secure-by-default behavior — not something anyone in the organization set. Don't assume a new Cloud PC behaves like established ones — see [Issue 2](#issue-2--layer-1-platform-default-disables-redirection).

### 2. Rule Out the Client and Group Policy

- **Client:** reproduce from a second machine, on a different OS, using the same client. If it fails the same way, the client and the connecting device's OS are ruled out.
- **Group Policy:** a local policy report with zero hits for anything clipboard-related means nothing is coming from Group Policy, local or domain.

See [Issue 1](#issue-1--obvious-explanations-ruled-out-first).

### 3. Configure All Three Settings in One Intune Policy

Put these together in a **single** policy:

- the **master clipboard-redirection switch** → allow
- the **local-device-to-remote-session** direction/data-type setting
- the **remote-session-to-local-device** direction/data-type setting

The two direction settings are independent of the master switch and both default to fully restricted — see [Issue 3](#issue-3--layer-2-separate-direction-and-format-restriction). Don't split the change across multiple profiles assigned to the same group — see [Issue 4](#issue-4--layer-3-self-inflicted-intune-policy-conflict).

### 4. Check the Device-Level Assignment Status

Use the **device-level assignment status report**, not just the policy's own summary. Look for:

- an explicit **conflict** on any device in the group
- a **pending** state that never resolves on the target device

### 5. Verify the Actual State on the Device

Read the real, current configuration directly off the device (registry) — not the management plane's summary of it. Expected: the master switch reads "allowed" **and** both direction-specific values are no longer "disabled".

Then test copy-paste in every direction: local → Cloud PC, Cloud PC → local, and Cloud PC → nested VM.

**If this all happens → clipboard redirection works in every direction.**

---

## Issues Encountered in This Case

### Issue 1 — Obvious explanations ruled out first

**Observed:** clipboard redirection completely broken in every direction — local device to Cloud PC, Cloud PC back to local device, and Cloud PC to a nested VM. Not "sometimes flaky" — completely dead, text included, let alone files.

**What didn't work:** treating it as a single on/off toggle and working through the usual suspects:

- **The remote desktop client's own clipboard setting** — no toggle found in the client being used. Dead end, especially once the same failure was reproduced from a second person's machine, on a different OS, using the same client. That ruled out the client and the connecting device's OS entirely.
- **A stray local or domain Group Policy** — a local policy report showed zero hits for anything clipboard-related.
- **The obvious Intune policy** — a registry check found the master clipboard-redirection switch explicitly set to *disabled*. Flipping it via Intune and syncing looked like the answer — but after the policy applied (confirmed via the device's own policy status report), clipboard still didn't work, in either direction, anywhere.

**Root cause:** three separate layers (Issues 2–4). The platform's own documentation explained the first two once the right question was asked: *why would a brand-new Cloud PC ship with this off by default at all?*

### Issue 2 — Layer 1: platform default disables redirection

**Observed:** master clipboard-redirection switch set to *disabled* on a brand-new Cloud PC, with no Group Policy setting it.

**Root cause:** Cloud PCs disable clipboard, drive, and printer redirection by default on first provisioning — a deliberate secure-by-default posture. New devices in this platform ship with different, more restrictive defaults than everything provisioned before a certain point. "This works everywhere else" was a reasonable-sounding argument that was actively wrong: the thing that changed wasn't the client, the network, or the user — it was that this device was new.

**Fix:** set the master switch to allow — necessary, but not sufficient (Issue 3).

### Issue 3 — Layer 2: separate direction-and-format restriction

**Observed:** after the master switch was allowed, clipboard still didn't work. A registry check showed the master switch read "allowed", while the direction-specific values still read "disabled", on both directions.

**Root cause:** a *separate* pair of settings controls clipboard **direction and allowed data types** — one for local-device-to-remote-session transfers, the other for the reverse — completely independent of the master switch. Both defaulted to fully restricted. Flipping the master switch doesn't touch them.

**Fix:** configure both direction settings explicitly as well ([step 3](#3-configure-all-three-settings-in-one-intune-policy)).

### Issue 4 — Layer 3: self-inflicted Intune policy conflict

**Observed:** with all settings identified and configured, Intune reported "succeeded" against the device — and the fix *still didn't apply*.

**Root cause:** the fix had been split across two separate Intune configuration profiles, both targeting overlapping settings in the same policy area, assigned to the same device group. The device-level assignment status report showed:

- on one device that shared the group — an explicit assignment **conflict**
- on the actual target device — a **pending** state for hours, never resolving, with no error surfaced anywhere that pointed at the real cause

Neither was visible from the policy's own status view.

**Fix:** consolidate everything — the master switch and both direction settings — into one single policy. Resolved immediately.

## Lessons Learned

- **A device redirection feature (clipboard, drives, printers, USB) can have more than one independent gate.** A master on/off switch and a granular direction/format restriction can exist side by side, and disabling one has zero effect on the other. Don't stop at the first setting that "sounds like" the whole feature.
- **"Policy applied successfully" is a claim about delivery, not about the resulting device state.** Always verify the actual, current configuration on the device itself after a policy reports success — especially before spending more time chasing a *different* theory for why a "successfully applied" fix isn't working.
- **When a correctly-configured setting still doesn't take effect, check for a conflicting assignment before assuming the setting itself is wrong.** A device-level assignment status report (not just the policy's own summary) is what actually surfaced this — it showed a conflict on one device and a stuck, unresolved pending state on the real target, neither of which was visible from the policy's own status view.
- **Consolidate related settings into one policy object when they're logically one change.** Splitting a single fix across multiple policy objects assigned to the same scope isn't just messier — it can create a genuine conflict that silently blocks the fix from ever landing, with no obvious error pointing at the real cause.
- **New provisioning ≠ same defaults as everything already in production.** Ruling out "it's broken for everyone" too early, based on how established devices behave, would have missed that this was a platform default specific to newly-provisioned devices.
- **The failure mode wasn't any single wrong setting — it was trusting a "succeeded" status** from a management platform as proof that the *intended* configuration is live on the device.

## Quick Reference Checklist

| If you see… | Do this |
|---|---|
| No clipboard in any direction on a new Cloud PC | Expected platform default — configure master switch **and** both direction settings |
| Same failure from another machine/OS with the same client | Client and local OS ruled out — look at device policy |
| Master switch "allowed" in registry, still no clipboard | Check the two direction-specific values — likely still "disabled" |
| Intune says "succeeded" but behavior unchanged | Read the registry on the device; check device-level assignment status |
| Assignment **conflict**, or **pending** for hours | Settings split across profiles on the same group — consolidate into one policy |
