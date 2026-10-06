# Building a Nested Proxmox Lab: Secure Boot Silently Falls Through to PXE

## Purpose of this Document

A case study and build reference for standing up Proxmox VE as a nested hypervisor inside a Hyper-V VM (itself running on a physical host). The build surfaced a chain of unrelated gotchas, each with a different kind of non-obvious cause — a capped RDP frame rate, a VM that wouldn't start because of a BIOS virtualization flag, a silent fallthrough to PXE caused by Secure Boot, and an installer that couldn't see its own DVD drive.

It is intentionally written to:

- Give the build sequence that worked, in order
- Keep that path separate from the gotchas hit along the way
- Record each gotcha with its symptom, cause and fix, so it can be recognised quickly next time
- Help identify which layer (physical host, outer VM, inner hypervisor) a symptom belongs to

## Environment

- Physical host with BIOS/UEFI firmware (Intel VT-x, Secure Boot)
- Hyper-V with nested virtualization
- Proxmox VE (9 attempted, 8.4 used) as the inner hypervisor
- Windows RDP for working inside the environment
- Intended use: a test environment eventually hosting internal test servers

---

## Correct End-to-End Process (Authoritative)

Nested virtualization (a hypervisor running inside a VM, which itself runs inside a physical host's hypervisor) means failures can originate at any of three layers — physical host, outer VM, or inner hypervisor — and it's not always obvious which one is responsible. Work through the layers in this order.

### 1. Fix the RDP Session First

A laggy RDP session makes every later diagnostic step slower and harder to read accurately. If the session is capped well below what the hardware should support, set the frame-interval registry value before going further — see [Issue 1](#issue-1--rdp-capped-at-30fps).

```
HKLM\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services
DWMFRAMEINTERVAL = <value>
```

### 2. Confirm VT-x Is Exposed From the Physical Host

Nested virtualization needs the processor's virtualization extensions exposed all the way through. "Hyper-V works for other VMs" is not enough.

If the nested VM won't start with a hypervisor-not-running error, check in order:

- `bcdedit` — hypervisor launch type set to auto
- Windows Optional Features — Hyper-V and its subcomponents present and enabled
- Core Isolation / Memory Integrity — can conflict with nested virtualization on some configurations
- **VT-x in the physical host's BIOS/UEFI** — must be enabled; needs physical access to the host

See [Issue 2](#issue-2--the-hypervisor-is-not-running).

### 3. Get Past Secure Boot

Proxmox's installer bootloader isn't signed in a way Secure Boot's default trusted-signature policy accepts. If the VM boots straight to a network PXE prompt with the ISO attached and first in the boot order, Secure Boot is rejecting the bootloader — see [Issue 3](#issue-3--silent-fallthrough-to-pxe-boot).

### 4. Use Proxmox VE 8.4, Not 9

Proxmox VE 9's graphical installer fails to detect the virtual DVD drive on Hyper-V. Proxmox VE 8.4 handles the emulated hardware correctly — see [Issue 4](#issue-4--installer-couldnt-see-its-own-dvd-drive).

### 5. Install via the Text-Mode Installer

The graphical installer intermittently loses keyboard input through nested console redirection. Use the installer's text-mode path — see [Issue 5](#issue-5--keyboard-input-swallowed-in-the-graphical-installer).

**If this all happens → Proxmox VE installed as a nested hypervisor.**

---

## Issues Encountered in This Case

### Issue 1 — RDP capped at 30fps

**Observed:** working inside the environment over RDP felt sluggish — capped well below what the hardware should support.

**Fix:** registry DWORD:

```
HKLM\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services
DWMFRAMEINTERVAL = <value>
```

Not related to anything downstream, but worth fixing early.

### Issue 2 — The hypervisor is not running

**Observed:** the nested VM refused to start with a hypervisor-not-running error — despite Hyper-V working correctly for other, non-nested VMs on the same physical host. That fact ruled out a Hyper-V installation-level problem and pointed at something specific to nested virtualization support.

**What didn't work:** checked in order, all fine:

- `bcdedit` — hypervisor launch type correctly set to auto
- Windows Optional Features — Hyper-V and its subcomponents all present and enabled
- Core Isolation / Memory Integrity — could conflict with nested virtualization on some configurations; checked and not the blocker here

**Root cause:** **VT-x (Intel virtualization extensions) was disabled in the physical host's BIOS/UEFI firmware.** Nested virtualization requires the processor's virtualization extensions exposed all the way through — a stricter requirement than "Hyper-V is enabled and working". A host can run ordinary Hyper-V VMs perfectly well with firmware configurations that still won't support a VM running its own inner hypervisor.

**Fix:** enable VT-x in the host BIOS/UEFI. This required physical access to the host — not resolvable remotely.

### Issue 3 — Silent fallthrough to PXE boot

**Observed:** with VT-x enabled and the VM starting, it booted straight into a network PXE boot prompt — despite an installer ISO being attached and correctly ordered first in the boot sequence. No error, no warning, just a clean skip past the ISO.

**Root cause:** **Secure Boot was silently rejecting Proxmox's installer bootloader** because it isn't signed in a way Secure Boot's default trusted-signature policy accepts. Rather than presenting a "this bootloader isn't trusted" error, UEFI firmware in this configuration treats the boot entry as if it doesn't exist and falls through to the next device in the boot order — which is what made it look like a boot-order or missing-media problem rather than a signature-trust problem.

**Fix:** get past the Secure Boot rejection of the bootloader; the install then proceeds to the installer.

### Issue 4 — Installer couldn't see its own DVD drive

**Observed:** past Secure Boot, Proxmox VE 9's graphical installer failed to detect the virtual DVD drive it had just booted from.

**Root cause:** a known compatibility issue between that installer version and Hyper-V's virtual IDE/SATA controller emulation.

**Fix:** downgrade to Proxmox VE 8.4 — the older installer's driver support handles the emulated hardware correctly.

### Issue 5 — Keyboard input swallowed in the graphical installer

**Observed:** once the installer loaded, keyboard input intermittently stopped reaching it inside the nested console session.

**Root cause:** an input-swallowing quirk of running a graphical installer through nested virtualization's console redirection.

**Fix:** switch to the installer's text-mode path, which didn't exhibit the issue.

## Lessons Learned

- **"Hyper-V works fine for other VMs" doesn't guarantee nested virtualization will work — VT-x exposure to the guest is a stricter, separate requirement,** checkable via BIOS/UEFI directly when `bcdedit` and Optional Features both look correct but the VM still won't start
- **A VM silently skipping past attached boot media straight to PXE, with no error, is a strong Secure Boot signature-rejection signal** — check it before assuming a boot-order or missing-ISO problem, especially for any non-Microsoft-signed OS installer
- **When an installer can't see its own boot media on Hyper-V, check for a known compatibility issue with that installer version before assuming a configuration mistake** — a version downgrade can be the actual fix, not a config change
- **Nested virtualization multiplies the number of layers a problem can originate from** (physical host firmware, outer hypervisor, inner OS/hypervisor, console redirection) — identify which layer a symptom belongs to before troubleshooting, rather than assuming it's always the innermost one

## Quick Reference Checklist

| If you see… | Do this |
|---|---|
| RDP session capped / sluggish | Set `DWMFRAMEINTERVAL` under `HKLM\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services` |
| Nested VM: "hypervisor is not running", but other Hyper-V VMs work | Check `bcdedit`, Optional Features, Core Isolation; then VT-x in physical host BIOS/UEFI |
| VM boots straight to PXE with ISO attached and first in boot order | Secure Boot rejecting the unsigned bootloader |
| Proxmox VE 9 installer can't see the virtual DVD drive on Hyper-V | Use Proxmox VE 8.4 |
| Keyboard input lost in graphical installer over nested console | Use the text-mode installer |
