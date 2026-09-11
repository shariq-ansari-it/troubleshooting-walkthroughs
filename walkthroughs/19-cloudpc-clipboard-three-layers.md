# The Cloud PC Clipboard That Needed Three Separate Fixes

**Stack:** Windows 365 Cloud PC, Microsoft Intune (Settings Catalog), Azure Virtual Desktop, nested Hyper-V

## TL;DR

Copy-paste didn't work at all on a brand-new Cloud PC — not from the local device into the Cloud PC, not the other way, and not into a nested VM running inside it either. The obvious fix (find the "allow clipboard redirection" policy and turn it on) got applied, reported success, and changed nothing. It took three separate, independently-gated layers to actually fix it: a master on/off switch, a completely separate direction-and-format restriction sitting on top of it, and — once both of those were identified — a self-inflicted Intune policy conflict that silently blocked the fix from ever reaching the device for hours after it was "configured correctly."

## The symptom

A newly provisioned Cloud PC had clipboard redirection completely broken in every direction: local device to Cloud PC, Cloud PC back to local device, and Cloud PC to a nested VM running inside it for testing. Not "sometimes flaky" — completely dead, text included, let alone files.

## Chasing the obvious explanations first

The instinct was to treat this as a single on/off toggle somewhere, and the early investigation worked through the usual suspects in order:

- **The remote desktop client's own clipboard setting.** No toggle was found in the client being used to connect — a dead end, especially once the same failure was reproduced from a second person's machine, on a different OS, using the same client. That ruled out the client and the connecting device's OS entirely.
- **A stray local or domain Group Policy.** A local policy report showed zero hits for anything clipboard-related — nothing was coming from Group Policy at all, local or domain.
- **The obvious Intune policy.** A registry check found the master clipboard-redirection switch explicitly set to *disabled*. That looked like the answer: flip it via Intune, sync, done.

Except after the policy applied — confirmed via the device's own policy status report — clipboard still didn't work, in either direction, anywhere.

## Getting to the real cause

The actual mechanism turned out to be three layers deep, and the platform's own documentation explained the first two once the right question was asked: *why would a brand-new Cloud PC ship with this off by default at all?*

**Layer 1 — the platform default.** Cloud PCs disable clipboard, drive, and printer redirection by default on first provisioning. This is documented, intentional behavior, not a misconfiguration — a deliberate secure-by-default posture, not something anyone in the organization had set.

**Layer 2 — a second, independent restriction on top of the first.** Fixing the master switch didn't fix anything, because there's a *separate* pair of settings controlling clipboard **direction and allowed data types** — one governing local-device-to-remote-session transfers, the other governing the reverse direction — completely independent of the master on/off switch. Both defaulted to fully restricted. Flipping the master switch to "allow" doesn't touch these at all; they have to be explicitly configured too. A quick registry check made this obvious once the two settings were known to look for: the master switch read "allowed," while the direction-specific values still read "disabled," on both directions.

**Layer 3 — a self-inflicted conflict.** With both settings identified, the fix was configured in Intune, reported as "succeeded" against the device, and *still didn't apply*. A device-level assignment status report showed why: the fix had been split across two separate Intune configuration profiles, both targeting overlapping settings in the same policy area, assigned to the same device group. On one device that shared the group, this showed up as an explicit assignment **conflict**; on the actual target device, it just sat in a **pending** state for hours, never resolving, with no error surfaced anywhere that pointed at the real cause. Consolidating everything — the master switch and both direction settings — into one single policy resolved it immediately.

## Why this matters going forward

The failure mode here wasn't any single wrong setting — it was trusting a "succeeded" status from a management platform as proof that the *intended* configuration is actually live on the device. A policy can apply cleanly, report success, and still not produce the described behavior, if it's silently losing an assignment-level conflict with another policy nobody thought to check for. The only way to actually confirm a fix landed was to read the real, current state directly off the device — not the management plane's summary of it.

The other lesson is about defaults: a brand-new managed endpoint should never be assumed to behave like an established one. "This works everywhere else" is a reasonable-sounding argument that was actively wrong here, because the thing that changed wasn't the client, the network, or the user — it was that this device was new, and new devices in this platform ship with different, more restrictive defaults than everything provisioned before a certain point.

## Takeaways

- **A device redirection feature (clipboard, drives, printers, USB) can have more than one independent gate.** A master on/off switch and a granular direction/format restriction can exist side by side, and disabling one has zero effect on the other. Don't stop at the first setting that "sounds like" the whole feature.
- **"Policy applied successfully" is a claim about delivery, not about the resulting device state.** Always verify the actual, current configuration on the device itself after a policy reports success — especially before spending more time chasing a *different* theory for why a "successfully applied" fix isn't working.
- **When a correctly-configured setting still doesn't take effect, check for a conflicting assignment before assuming the setting itself is wrong.** A device-level assignment status report (not just the policy's own summary) is what actually surfaced this — it showed a conflict on one device and a stuck, unresolved pending state on the real target, neither of which was visible from the policy's own status view.
- **Consolidate related settings into one policy object when they're logically one change.** Splitting a single fix across multiple policy objects assigned to the same scope isn't just messier — it can create a genuine conflict that silently blocks the fix from ever landing, with no obvious error pointing at the real cause.
- **New provisioning ≠ same defaults as everything already in production.** Ruling out "it's broken for everyone" too early, based on how established devices behave, would have missed that this was a platform default specific to newly-provisioned devices.
