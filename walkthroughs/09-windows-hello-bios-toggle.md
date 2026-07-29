# Three Days Chasing an Intune Policy — the Real Cause Was a BIOS Toggle

**Stack:** Windows Hello for Business, Microsoft Intune (Settings Catalog), Azure Virtual Desktop, laptop BIOS/UEFI

## TL;DR

Windows Hello wouldn't set up on a brand-new laptop, and every symptom pointed at an Intune Windows Hello for Business / Convenience PIN policy problem — scoping, assignment groups, per-setting "Not applicable" states, even a plausible-sounding theory about WHfB conflicting with AVD password resets tenant-wide. Three layers into policy archaeology, all of it was wrong. The actual on-screen error message — never actually read carefully until day three — said Windows couldn't find a compatible camera or fingerprint scanner. The real cause was a BIOS setting called "Passwordless authentication," silently switched off, and non-responsive to clicking because the vendor's firmware requires a BIOS supervisor password to be set before that toggle can be changed at all.

## Why the investigation went the way it did

The initial assumption was reasonable: new laptop, Windows Hello won't configure, this is an Intune Windows Hello for Business or Convenience PIN deployment issue. That's a common enough real failure mode, so it's where the investigation started — and stayed, for a while.

## What got investigated (all correct-sounding, all wrong)

- **Settings Catalog assignment scoping** — confirmed the WHfB/PIN policy was assigned to the right device group, with no obvious group-membership gap.
- **Per-setting policy report states** — checked whether individual settings inside the profile were reporting "Succeeded" vs. "Not applicable" per device, looking for a setting silently not landing.
- **A tenant-wide WHfB interaction theory** — considered whether Windows Hello for Business policy was somehow conflicting with password-reset behavior on Azure Virtual Desktop hosts elsewhere in the tenant, since that's a real, documented category of interaction problem in mixed AVD/WHfB environments.

All of this was directionally sensible troubleshooting for "Windows Hello won't configure on an Intune-managed device." None of it was the actual cause on this specific laptop.

## The turning point

The actual on-screen message, once read closely instead of skimmed past as a generic "can't set up Windows Hello" notice, said something to the effect of: *"We couldn't find a camera or fingerprint scanner compatible with Windows Hello."* That is not a policy-denial message — Intune-blocked WHfB configuration shows a different, policy-flavored error. This was Windows itself reporting it couldn't see usable biometric hardware.

## Root cause

The laptop (a Samsung Galaxy Book5) had a BIOS setting labeled "Passwordless authentication" switched **Off**. Attempting to toggle it in the BIOS UI did nothing — it wasn't greyed out or disabled-looking, it just silently refused to respond to input, which is exactly the kind of behavior that makes you assume you must be missing some other setting rather than suspect this one. The actual gate: Samsung's firmware requires a **BIOS supervisor password** to be configured first, before that specific toggle becomes editable at all. No supervisor password set → the passwordless-auth toggle is present, visible, and completely inert.

Once a supervisor password was set in BIOS, the toggle became editable, was switched on, and Windows Hello enrollment worked immediately with no Intune-side changes of any kind.

## Takeaways

- **Read the exact on-screen error text before building a theory, not after.** "Windows Hello won't set up" was skimmed as one generic failure class for the first two investigation days; the actual message was specific and pointed somewhere completely different from where the effort was going.
- **A hardware-detection error and a policy-denial error can look similar at a glance but are diagnostically unrelated** — one says "there's no usable sensor," the other says "policy says no." Confirm which one you're actually looking at before choosing an investigation path.
- **Vendor BIOS/UEFI settings can be silently gated behind an unrelated prerequisite** (here, a supervisor password) with zero indication in the UI other than the control simply not responding to being toggled. If a BIOS setting won't change and gives no error, check whether the vendor requires something else configured first.
- **A plausible cross-system theory (WHfB vs. AVD password resets) is still worth ruling out — but rule it out fast and move to reading the actual client-side error, rather than letting a plausible theory anchor the investigation for days.**
