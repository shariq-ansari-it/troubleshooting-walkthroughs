# Three Days Chasing an Intune Policy — the Real Cause Was a BIOS Toggle

## TL;DR

Windows Hello wouldn't set up on a brand-new laptop, and every symptom pointed at an Intune Windows Hello for Business / Convenience PIN policy problem — scoping, assignment groups, per-setting "Not applicable" states, even a plausible-sounding theory about WHfB conflicting with AVD password resets tenant-wide. Three layers into policy archaeology, all of it was wrong. The actual on-screen error message — never actually read carefully until day three — said Windows couldn't find a compatible camera or fingerprint scanner. The real cause was a BIOS setting called "Passwordless authentication," silently switched off, and non-responsive to clicking because the vendor's firmware requires a BIOS supervisor password to be set before that toggle can be changed at all.

## Purpose of this Document

A case study and reference runbook for the problem summarised in the TL;DR above.

It is intentionally written to:

- Give the path that actually resolved it, so it can be repeated in minutes rather than days
- Keep that path separate from the policy theories that were investigated first
- Record each dead end with why it looked right and why it wasn't, so the pattern is recognised quickly next time
- Reinforce reading the exact on-screen error before building a theory

## Environment

- Windows Hello for Business / Convenience PIN
- Microsoft Intune (Settings Catalog)
- Azure Virtual Desktop elsewhere in the tenant
- Laptop BIOS/UEFI — Samsung Galaxy Book5

---

## Correct End-to-End Process (Authoritative)

### 1. Read the Exact On-Screen Error

Read the Windows Hello setup message closely, not as a generic "can't set up Windows Hello" notice. Here it said, to the effect of:

```
We couldn't find a camera or fingerprint scanner compatible with Windows Hello.
```

### 2. Decide: Hardware-Detection or Policy-Denial?

- **Hardware-detection error** — Windows itself reporting it can't see usable biometric hardware (the message above)
- **Policy-denial error** — Intune-blocked WHfB configuration shows a different, policy-flavored error

This message is not a policy denial, so go to the device, not Intune (see [Issue 3](#issue-3--the-error-message-was-skimmed-not-read)).

### 3. Check the BIOS "Passwordless authentication" Setting

In BIOS/UEFI, find **Passwordless authentication**. In this case it was **Off**, and toggling it did nothing (see [Issue 4](#issue-4--bios-toggle-silently-refuses-to-change)).

### 4. Set a BIOS Supervisor Password

The vendor's firmware requires a **BIOS supervisor password** to be configured before that toggle becomes editable at all.

### 5. Switch Passwordless Authentication On and Enroll

With the supervisor password set, the toggle became editable and was switched **On**. Windows Hello enrollment then worked immediately, with no Intune-side changes of any kind.

**If this all happens → Windows Hello enrolls.**

---

## Issues Encountered in This Case

### Issue 1 — Investigation anchored on the Intune WHfB/PIN policy

**Observed:** new laptop, Windows Hello won't configure — assumed to be an Intune Windows Hello for Business or Convenience PIN deployment issue. That's a common enough real failure mode, so it's where the investigation started, and stayed.

**What didn't work:**

- **Settings Catalog assignment scoping** — confirmed the WHfB/PIN policy was assigned to the right device group, with no obvious group-membership gap.
- **Per-setting policy report states** — checked whether individual settings inside the profile were reporting "Succeeded" vs "Not applicable" per device, looking for a setting silently not landing.

**Root cause:** all directionally sensible troubleshooting for "Windows Hello won't configure on an Intune-managed device" — none of it was the cause on this laptop. See [step 3](#3-check-the-bios-passwordless-authentication-setting).

### Issue 2 — Tenant-wide WHfB vs AVD password-reset theory

**Observed:** considered whether Windows Hello for Business policy was conflicting with password-reset behavior on Azure Virtual Desktop hosts elsewhere in the tenant — a real, documented category of interaction problem in mixed AVD/WHfB environments.

**Root cause:** plausible, but not the cause here. Worth ruling out — fast — rather than letting it anchor the investigation for days.

### Issue 3 — The error message was skimmed, not read

**Observed:** for the first two investigation days, "Windows Hello won't set up" was treated as one generic failure class. The actual message — never read carefully until day three — said Windows couldn't find a compatible camera or fingerprint scanner.

**Root cause:** a hardware-detection error and a policy-denial error look similar at a glance but are diagnostically unrelated.

**Fix:** read the exact text first ([step 1](#1-read-the-exact-on-screen-error)).

### Issue 4 — BIOS toggle silently refuses to change

**Observed:** "Passwordless authentication" in BIOS was **Off**. Attempting to toggle it did nothing — not greyed out or disabled-looking, it just silently refused to respond to input. That behavior makes you assume you're missing some other setting rather than suspect this one.

**Root cause:** the vendor's firmware requires a **BIOS supervisor password** to be configured first. No supervisor password set → the passwordless-auth toggle is present, visible, and completely inert.

**Fix:** set a supervisor password, then switch the toggle on ([step 4](#4-set-a-bios-supervisor-password)).

## Lessons Learned

- **Read the exact on-screen error text before building a theory, not after.** "Windows Hello won't set up" was skimmed as one generic failure class for the first two investigation days; the actual message was specific and pointed somewhere completely different from where the effort was going.
- **A hardware-detection error and a policy-denial error can look similar at a glance but are diagnostically unrelated** — one says "there's no usable sensor," the other says "policy says no." Confirm which one you're actually looking at before choosing an investigation path.
- **Vendor BIOS/UEFI settings can be silently gated behind an unrelated prerequisite** (here, a supervisor password) with zero indication in the UI other than the control simply not responding to being toggled. If a BIOS setting won't change and gives no error, check whether the vendor requires something else configured first.
- **A plausible cross-system theory (WHfB vs. AVD password resets) is still worth ruling out — but rule it out fast and move to reading the actual client-side error, rather than letting a plausible theory anchor the investigation for days.**

## Quick Reference Checklist

| If you see… | Do this |
|---|---|
| Windows Hello won't set up | Read the exact on-screen message before touching Intune |
| "Couldn't find a camera or fingerprint scanner compatible with Windows Hello" | Hardware-detection problem, not policy — check BIOS |
| Policy-flavored error | Then investigate the Intune WHfB/PIN policy |
| BIOS "Passwordless authentication" Off and won't toggle | Set a BIOS supervisor password first, then switch it on |
| Any BIOS setting that won't change and gives no error | Check whether the vendor requires another setting configured first |
