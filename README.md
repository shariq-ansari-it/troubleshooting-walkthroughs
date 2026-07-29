# Troubleshooting Walkthroughs

Write-ups of real infrastructure problems I've diagnosed and fixed — mostly Azure, Microsoft 365/Intune, and general Windows server administration. Details are generalized (no company names, internal hostnames, ticket numbers, or account identifiers), but the technical substance — the symptoms, the dead ends, the root cause, and the fix — is real.

## Index

1. [The Sponsorship Credit That Wouldn't Apply](walkthroughs/01-billing-credit-wrong-account.md) — an Azure subscription kept getting billed in full despite a redeemed credit; the credit and the subscription turned out to be tied to two different billing accounts, and the standard support paths for fixing it were all dead ends.
2. [Patching a VDI Fleet Without Touching Everyone Else's Session](walkthroughs/02-vdi-fleet-patch-targeting.md) — Intune supersedence and Assignment Filters to target one misbehaving VM, plus an undocumented Graph API quirk.
3. [The Firewall Change That Broke a Customer We Didn't Touch](walkthroughs/03-firewall-change-broke-customer.md) — a network security change to an internal-only server had a side effect on a completely external, customer-facing endpoint.
4. [Guest Account Sprawl From "Helpful" Sharing Links](walkthroughs/04-guest-account-sprawl.md) — tracing an identity sprawl problem back to a sharing feature, and the security tradeoff behind the fix.
5. [Automating a VM Patch Window With Azure Automation + Resource Graph](walkthroughs/05-vm-patch-window-automation.md) — a small self-service automation system for a recurring maintenance chore.
6. [Rolling Out MFA to Servers With No Directory Service](walkthroughs/06-mfa-rollout-no-directory.md) — enforcing modern authentication on machines that have no AD, no SSO, and nothing to sync from.
7. [Why `az login` Can't Query Intune (And What Does)](walkthroughs/07-az-login-cant-query-intune.md) — a sharp, easy-to-hit Azure CLI authentication gotcha and its fix.
8. [The BitLocker "Access Denied" That Wasn't About BitLocker's Own Settings](walkthroughs/08-bitlocker-hidden-intune-policy.md) — write-protection on an external drive traced to an Intune policy buried in a generically-named Settings Catalog profile.
9. [Three Days Chasing an Intune Policy — the Real Cause Was a BIOS Toggle](walkthroughs/09-windows-hello-bios-toggle.md) — Windows Hello setup failures that had nothing to do with Intune at all.
10. [Stuck at "Registered," Not "Joined" — the Fix Was a Link, Not a Network Fix](walkthroughs/10-registered-not-joined.md) — a WinInet error sent the investigation down a network rabbit hole; the real fix was a different button.
11. [Building a Nested Proxmox Lab: Secure Boot Silently Falls Through to PXE](walkthroughs/11-nested-proxmox-secure-boot-pxe.md) — five unrelated gotchas in one nested-virtualization build.
12. [Windows OpenSSH Can't Authenticate a Cloud-Only Entra Identity](walkthroughs/12-openssh-entra-only-identity.md) — a genuinely surprising, documented Win32-OpenSSH limitation, plus a few smaller SSH-on-Windows surprises.
13. [Three "Noncompliant" Devices, Three Unrelated Root Causes](walkthroughs/13-three-noncompliant-devices.md) — disproving a shared-cause theory, and finding a device that had silently stopped syncing for months while in daily use.
14. [Migrating a Golden-Copy VHD Into Azure Across Regions](walkthroughs/14-cross-region-vhd-migration.md) — two separate disk-import restrictions, and a staged blob-copy workaround.
15. [An RDS Environment's Performance Collapse, Triggered by Onboarding](walkthroughs/15-rds-performance-collapse.md) — new users weren't the cause, they were what finally exposed three compounding problems already there.
16. [An Orphaned Subscription Blocked by Two Independently Deprecated Transfer Paths](walkthroughs/16-orphaned-subscription-two-deprecated-paths.md) — when a legacy Azure subscription ages past what any self-service tool can fix.
