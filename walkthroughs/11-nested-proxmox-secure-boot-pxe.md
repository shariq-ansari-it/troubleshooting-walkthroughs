# Building a Nested Proxmox Lab: Secure Boot Silently Falls Through to PXE

**Stack:** Hyper-V (nested virtualization), Proxmox VE, BIOS/UEFI (VT-x, Secure Boot), Windows RDP

## TL;DR

Building a Proxmox VE test environment inside a Hyper-V VM (itself running on a physical host) surfaced a chain of unrelated gotchas, each with a different kind of non-obvious cause: a capped RDP frame rate fixed by a registry value, a VM that wouldn't start at all traced to a BIOS virtualization flag despite Hyper-V already working for other VMs, a VM that booted straight to network PXE with zero error message because Secure Boot was silently rejecting Proxmox's unsigned bootloader, and an installer that couldn't see its own attached virtual DVD drive — a known Hyper-V compatibility bug requiring a version downgrade.

## Goal

Stand up Proxmox VE as a nested hypervisor inside Hyper-V, as a test environment eventually intended to host internal test servers. Nested virtualization (a hypervisor running inside a VM, which itself runs inside a physical host's hypervisor) is exactly the kind of setup where failures can originate at any of three layers — physical host, outer VM, or inner hypervisor — and it's not always obvious which one is actually responsible.

## Gotcha 1: RDP capped at 30fps

Working inside the environment over RDP felt sluggish — capped well below what the hardware should support. Fixed with a registry DWORD:

```
HKLM\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services
DWMFRAMEINTERVAL = <value>
```

Not related to anything downstream, but worth fixing early since a laggy RDP session makes every subsequent diagnostic step slower and harder to read accurately.

## Gotcha 2: "The hypervisor is not running"

The nested VM refused to start at all with a hypervisor-not-running error — despite Hyper-V clearly working correctly for other, non-nested VMs on the same physical host. That "it works for everything else" fact ruled out a Hyper-V-installation-level problem and pointed at something specific to nested virtualization support.

Checked, in order:
- `bcdedit` — hypervisor launch type correctly set to auto.
- Windows Optional Features — Hyper-V and its subcomponents all present and enabled.
- Core Isolation / Memory Integrity — could conflict with nested virtualization on some configurations; checked and not the blocker here.

Root cause: **VT-x (Intel virtualization extensions) was disabled in the physical host's BIOS/UEFI firmware.** Nested virtualization requires the *processor's* virtualization extensions exposed all the way through, which is a stricter requirement than "Hyper-V is enabled and working" — a host can run ordinary Hyper-V VMs perfectly well with certain firmware configurations that still won't support a VM running its own inner hypervisor. Fixing it required physical access to the host to flip the BIOS setting — not something resolvable remotely.

## Gotcha 3: Silent fallthrough to PXE boot

With VT-x enabled and the VM starting, it then booted straight into a network PXE boot prompt — despite an installer ISO being attached and correctly ordered first in the boot sequence. No error, no warning, just a clean skip past the ISO straight to the network boot attempt.

Root cause: **Secure Boot was silently rejecting Proxmox's installer bootloader** because it isn't signed in a way Secure Boot's default trusted-signature policy accepts. Rather than presenting a "this bootloader isn't trusted" error, UEFI firmware in this configuration just treats the boot entry as if it doesn't exist and falls through to the next device in the boot order — which is exactly what made this look like a boot-order or missing-media problem rather than a signature-trust problem.

## Gotcha 4: Installer couldn't see its own DVD drive

Past Secure Boot, Proxmox VE 9's graphical installer failed to detect the virtual DVD drive it had just booted from — a known compatibility issue between that installer version and Hyper-V's virtual IDE/SATA controller emulation. Downgrading to Proxmox VE 8.4 resolved it outright; the older installer's driver support handles the emulated hardware correctly.

## Gotcha 5: Keyboard input swallowed in the graphical installer

Once the installer loaded, keyboard input intermittently stopped reaching it inside the nested console session — an input-swallowing quirk of running a graphical installer through nested virtualization's console redirection. Worked around by switching to the installer's text-mode path instead of the graphical one, which didn't exhibit the issue.

## Takeaways

- **"Hyper-V works fine for other VMs" doesn't guarantee nested virtualization will work — VT-x exposure to the guest is a stricter, separate requirement,** checkable via BIOS/UEFI directly when `bcdedit` and Optional Features both look correct but the VM still won't start.
- **A VM silently skipping past attached boot media straight to PXE, with no error, is a strong Secure Boot signature-rejection signal** — worth checking before assuming a boot-order or missing-ISO problem, especially for any non-Microsoft-signed OS installer.
- **When an installer can't see its own boot media on Hyper-V, check for a known compatibility issue with that specific installer version before assuming a configuration mistake** — a version downgrade can be the actual fix, not a config change.
- **Nested virtualization multiplies the number of layers a problem can originate from** (physical host firmware, outer hypervisor, inner OS/hypervisor, console redirection) — worth deliberately identifying which layer a symptom belongs to before troubleshooting, rather than assuming it's always the innermost one.
