# The BitLocker "Access Denied" That Wasn't About BitLocker's Own Settings

## Purpose of this Document

A case study for an external USB hard drive that suddenly started throwing "the media is write protected" on an Entra-joined Windows 11 PC, for a user who was a local administrator. Every classic cause checked out clean; the actual cause was an Intune-enforced policy requiring BitLocker on removable drives, buried in a generically-named Settings Catalog profile rather than under the Disk Encryption blade.

It is intentionally written to:

- Give the order of checks that actually finds the cause on an Intune-managed device
- Keep that path separate from the classic write-protection checks that all came back clean
- Record each dead end with its symptom and why it wasn't the cause, so it can be skipped or ruled out quickly next time
- Help recognise the "hidden policy in a generically-named profile" pattern

## Environment

- Windows 11, Entra-joined, Intune-managed
- BitLocker (removable drives)
- Microsoft Intune — Settings Catalog profiles, Endpoint Security → Disk Encryption
- External USB hard drives
- Affected user is a local administrator

---

## Correct End-to-End Process (Authoritative)

### 1. Rule Out the Drive Early

Try a second, completely different external drive. If it fails identically, it isn't the hardware — move on to policy rather than spending more time on the drive.

### 2. Quick-Check the Classic Causes

These were all clean in this case (see [Issue 1](#issue-1--media-is-write-protected-with-every-classic-cause-clean)), but each is fast:

- `diskpart` → `select disk` → `attributes disk` — disk-level read-only flag
- `HKLM\SYSTEM\CurrentControlSet\Control\StorageDevicePolicies` — legacy `WriteProtect` value
- `icacls` on the drive — NTFS permissions

Skip Local Group Policy Editor on an Entra-joined device — see [Issue 2](#issue-2--local-group-policy-doesnt-show-what-intune-enforces).

### 3. Check the CSP-Delivered BitLocker Policy Directly

Look in:

```
HKLM\SOFTWARE\Microsoft\PolicyManager\current\device\BitLocker
```

In this case:

```
RemovableDrivesRequireEncryption_ProviderSet = 1
```

— a CSP-delivered policy requiring BitLocker encryption before a removable drive can be written to. This is also why the "Turn on BitLocker" wizard fails — see [Issue 4](#issue-4--turn-on-bitlocker-wizard-also-fails-with-access-denied).

### 4. Find the Profile That Delivers It

Don't stop at **Endpoint Security → Disk Encryption** — the setting may not be there (see [Issue 5](#issue-5--policy-not-visible-under-the-disk-encryption-blade)). Search **all** Settings Catalog profiles assigned to the device for the BitLocker CSP area, including generically-named ones.

**If this all happens → the enforcing policy and the profile delivering it are identified.**

---

## Issues Encountered in This Case

### Issue 1 — "Media is write protected" with every classic cause clean

**Observed:** copying files onto an external HDD failed immediately with the classic "media is write protected" message — which normally means a physical write-lock switch, a `diskpart` read-only attribute, or a permissions problem.

**What didn't work** (checked in order):

1. **Disk-level write-protect flag** — `diskpart` → `select disk` → `attributes disk` showed nothing set.
2. **`StorageDevicePolicies\WriteProtect` registry value** — a known legacy way to flip removable media to read-only tenant-wide. `HKLM\SYSTEM\CurrentControlSet\Control\StorageDevicePolicies` — not present.
3. **NTFS permissions** — `icacls` on the drive confirmed the user had Full Control via the local Administrators group. Not a permissions problem in the conventional sense.
4. **Hardware** — a second, completely different external HDD gave the identical failure. Ruled out a bad drive.

**Root cause:** MDM-enforced encryption requirement, not write protection — see [step 3](#3-check-the-csp-delivered-bitlocker-policy-directly).

### Issue 2 — Local Group Policy doesn't show what Intune enforces

**Observed:** Local Group Policy is a natural next check for a policy-driven restriction.

**Root cause:** on an Entra-joined, Intune-managed device, policies arrive as MDM/CSP configuration, not classic GPOs — the Local Group Policy Editor wouldn't even see what's actually being enforced.

**Fix:** go to the `PolicyManager` registry and the Intune profiles instead.

### Issue 3 — `BitLocker-Driver` event in Event Viewer was a red herring

**Observed:** a `BitLocker-Driver` event was present around the time of the failure — looked promising.

**Root cause:** routine `fvevol.sys` (the BitLocker filter driver) telemetry, not an error.

### Issue 4 — "Turn on BitLocker" wizard also fails with Access Denied

**Observed:** trying BitLocker's own "Turn on BitLocker" wizard directly on the drive failed with Access Denied.

**Root cause:** the same policy. Two settings were effectively fighting each other from the user's point of view: one policy blocking unencrypted writes, and — depending on how the Settings Catalog profile combined its options — the encryption workflow itself not completing the way it normally would through the Windows UI. From the user's side, the drive flatly refused both being written to and being fixed.

### Issue 5 — Policy not visible under the Disk Encryption blade

**Observed:** nothing relevant under Intune's dedicated **Endpoint Security → Disk Encryption** blade, where you'd naturally look for a BitLocker-related setting.

**Root cause:** the setting had been configured as part of a broader Settings Catalog profile with a generic name (something like "Security Enhancements") that bundled multiple unrelated hardening settings together — BitLocker enforcement being one line item buried inside a profile whose name gave no hint that BitLocker was in scope at all.

**Fix:** search all Settings Catalog profiles assigned to the device for the relevant CSP area ([step 4](#4-find-the-profile-that-delivers-it)).

## Lessons Learned

- **"Access Denied" on removable media isn't always about NTFS or physical write-protection — it can be an MDM-enforced encryption requirement blocking unencrypted writes outright.** `HKLM\SOFTWARE\Microsoft\PolicyManager\current\device\BitLocker` is worth checking directly rather than assuming the Disk Encryption blade shows everything BitLocker-related.
- **On an Entra-joined/Intune-managed device, don't reach for Local Group Policy Editor as a diagnostic step.** Policy is arriving via CSPs through MDM, and the local GPO view won't reflect it.
- **Generically-named Settings Catalog profiles are a real discoverability problem.** A profile named for a broad theme ("Security Enhancements," "Baseline Hardening," etc.) can bury a specific, high-impact setting somewhere no one would think to look for it. When troubleshooting an Intune-enforced behavior, search *all* Settings Catalog profiles assigned to the device for the relevant CSP area, not just the blade that seems purpose-built for it.
- **A hardware swap is a fast, cheap way to rule out "maybe it's this specific drive"** before spending more time on policy archaeology — worth doing early, not late.

## Quick Reference Checklist

| If you see… | Do this |
|---|---|
| "Media is write protected" on a removable drive, Intune-managed device | Check `HKLM\SOFTWARE\Microsoft\PolicyManager\current\device\BitLocker` |
| `RemovableDrivesRequireEncryption_ProviderSet = 1` | Intune is requiring BitLocker on removable drives — find the profile delivering it |
| Nothing under Endpoint Security → Disk Encryption | Search all assigned Settings Catalog profiles for the BitLocker CSP area |
| "Turn on BitLocker" wizard → Access Denied | Same policy — not a separate problem |
| `BitLocker-Driver` event in Event Viewer | Likely routine `fvevol.sys` telemetry, not an error |
| Unsure whether it's the drive | Try a different drive first |
| Reaching for Local Group Policy Editor on an Entra-joined device | Don't — policy arrives via MDM/CSP |
