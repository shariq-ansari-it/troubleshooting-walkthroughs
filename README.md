# Troubleshooting Walkthroughs

Write-ups of real infrastructure problems I've diagnosed and fixed — mostly Azure, Microsoft 365/Intune, and general Windows server administration. Details are generalized (no company names, internal hostnames, ticket numbers, or account identifiers), but the technical substance — the symptoms, the dead ends, the root cause, and the fix — is real.

## Index

1. [The Sponsorship Credit That Wouldn't Apply](walkthroughs/01-billing-credit-wrong-account.md) — an Azure subscription kept getting billed in full despite a redeemed credit; the credit and the subscription turned out to be tied to two different billing accounts, and the standard support paths for fixing it were all dead ends.
2. [Patching a VDI Fleet Without Touching Everyone Else's Session](walkthroughs/02-vdi-fleet-patch-targeting.md) — Intune supersedence and Assignment Filters to target one misbehaving VM, plus an undocumented Graph API quirk.
3. [The Firewall Change That Broke a Customer We Didn't Touch](walkthroughs/03-firewall-change-broke-customer.md) — a network security change to an internal-only server had a side effect on a completely external, customer-facing endpoint.
4. [Guest Account Sprawl From "Helpful" Sharing Links](walkthroughs/04-guest-account-sprawl.md) — tracing an identity sprawl problem back to a sharing feature, and the security tradeoff behind the fix.
5. [Automating a VM Patch Window With Azure Automation + Resource Graph](walkthroughs/05-vm-patch-window-automation.md) — a small self-service automation system for a recurring maintenance chore.
6. [Rolling Out MFA to Servers With No Directory Service](walkthroughs/06-mfa-rollout-no-directory.md) — enforcing modern authentication on machines that have no AD, no SSO, and nothing to sync from.
7. Why `az login` Can't Query Intune (And What Does) — a sharp, easy-to-hit Azure CLI authentication gotcha and its fix.

*(Items above without a link are on the way.)*
