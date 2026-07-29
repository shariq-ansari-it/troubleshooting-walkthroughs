# The BitLocker "Access Denied" That Wasn't About BitLocker's Own Settings

**Stack:** Windows 11, BitLocker, Microsoft Intune (Settings Catalog), Entra-joined device management

## TL;DR

An external USB hard drive suddenly started throwing "the media is write protected" on an Entra-joined Windows 11 PC, for a user who was a local administrator. Every classic cause for that message checked out clean — disk-level flags, NTFS permissions, even the hardware itself (same result on a second, different drive). The actual cause was an Intune-enforced policy requiring BitLocker encryption on removable drives, which also blocked BitLocker's own encryption wizard from running — and which wasn't visible under Intune's dedicated Disk Encryption blade at all, because it had been bundled into a generically-named Settings Catalog profile instead.

## The symptom

Trying to copy files onto an external HDD failed immediately with a write-protection error — the classic "media is write protected" message that normally means a physical write-lock switch, a `diskpart` read-only attribute, or a permissions problem. None of the classic causes applied here.

## Working through the usual suspects

In order:

1. **Disk-level write-protect flag.** `diskpart` → `select disk` → `attributes disk` showed nothing set. Not this.
2. **`StorageDevicePolicies\WriteProtect` registry value.** A known legacy way to flip removable media to read-only tenant-wide. Checked `HKLM\SYSTEM\CurrentControlSet\Control\StorageDevicePolicies` — not present.
3. **Local Group Policy.** Irrelevant on an Entra-joined, Intune-managed device — policies here arrive as MDM/CSP configuration, not classic GPOs, so a Local Group Policy Editor check wouldn't even see what's actually being enforced.
4. **NTFS permissions.** `icacls` on the drive confirmed the user had Full Control via the local Administrators group. Not a permissions problem in the conventional sense.
5. **Event Viewer.** A `BitLocker-Driver` event was present around the time of the failure — looked promising, turned out to just be routine `fvevol.sys` (the BitLocker filter driver) telemetry, not an error.
6. **Hardware.** Tried a second, completely different external HDD. Identical failure. Ruled out a bad drive.

## Root cause

The actual policy was sitting in `HKLM\SOFTWARE\Microsoft\PolicyManager\current\device\BitLocker`, specifically `RemovableDrivesRequireEncryption_ProviderSet = 1` — a CSP-delivered policy requiring BitLocker encryption before a removable drive can be written to.

The twist: this same policy is *also* why BitLocker's own "Turn on BitLocker" wizard failed with Access Denied when tried directly on the drive. Two settings were effectively fighting each other from the user's point of view: one policy blocking unencrypted writes, and — depending on how the Settings Catalog profile combined its options — the encryption workflow itself not completing the way it normally would through the Windows UI. From the user's side, it just looked like the drive flatly refused both reading and being fixed.

This policy was not visible under Intune's dedicated **Endpoint Security → Disk Encryption** blade, where you'd naturally look for a BitLocker-related setting. It had been configured as part of a broader Settings Catalog profile with a generic name (something like "Security Enhancements") that bundled multiple unrelated hardening settings together — BitLocker enforcement being just one line item buried inside a profile whose name gave no hint that BitLocker was in scope at all.

## Takeaways

- **"Access Denied" on removable media isn't always about NTFS or physical write-protection — it can be an MDM-enforced encryption requirement blocking unencrypted writes outright.** `HKLM\SOFTWARE\Microsoft\PolicyManager\current\device\BitLocker` is worth checking directly rather than assuming the Disk Encryption blade shows everything BitLocker-related.
- **On an Entra-joined/Intune-managed device, don't reach for Local Group Policy Editor as a diagnostic step.** Policy is arriving via CSPs through MDM, and the local GPO view won't reflect it.
- **Generically-named Settings Catalog profiles are a real discoverability problem.** A profile named for a broad theme ("Security Enhancements," "Baseline Hardening," etc.) can bury a specific, high-impact setting somewhere no one would think to look for it. When troubleshooting an Intune-enforced behavior, search *all* Settings Catalog profiles assigned to the device for the relevant CSP area, not just the blade that seems purpose-built for it.
- **A hardware swap is a fast, cheap way to rule out "maybe it's this specific drive"** before spending more time on policy archaeology — worth doing early, not late.
