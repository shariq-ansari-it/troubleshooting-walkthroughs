# `Connect-ExchangeOnline` Throws `PlatformNotSupportedException` on a New macOS Release

## TL;DR

`Connect-ExchangeOnline` (interactive, browser-based auth) can fail immediately with a `PlatformNotSupportedException` on a recently released macOS version — before any login prompt appears, and with no network or account problem involved. The exception message is literally the macOS version string. This is MSAL's cross-platform browser-launch code failing to recognize how new the OS is, not anything wrong with your credentials, your tenant, or your machine. The fix is to skip the local-browser launch path entirely with device code authentication.

## Purpose of this Document

A case study and reference runbook for the problem summarised in the TL;DR above.

It is intentionally written to:

- Give the working connection method, so an Exchange Online session can be opened without troubleshooting the local environment
- Explain the root cause, so the error isn't misread as a permissions, network or module-installation problem
- Help recognize the same failure in other MSAL-backed PowerShell modules

## Environment

- macOS (newly released version — `macOS 26.6.2` in this case)
- PowerShell 7
- ExchangeOnlineManagement (EXO V3)
- MSAL.NET (bundled with the module)

### Background: How This Came Up

The task was routine: move a secondary email address from one shared mailbox to another, so mail sent to that address lands in a different inbox. An address like that isn't necessarily its own object — it can just as easily be a proxy address (an alias) hanging off an unrelated mailbox, and there's no way to tell from the address alone. Searching for it in the admin center as if it were its own mailbox turns up nothing, which reads like the address doesn't exist at all.

The only reliable way to resolve it is to query Exchange Online directly and see what recipient the address resolves to:

```powershell
Get-Recipient -Identity thataddress@example.com | Select-Object DisplayName, RecipientType, RecipientTypeDetails, PrimarySmtpAddress, EmailAddresses
```

That confirmed the address was a secondary `smtp:` proxy address on a different mailbox entirely, not a mailbox of its own — which is also what made the fix safe: removing it from one mailbox's `EmailAddresses` collection and adding it to the other's via `Set-Mailbox`, rather than anything involving forwarding rules or a real second mailbox object.

Getting there needed an interactive Exchange Online PowerShell session — and that's where the gotcha below showed up, before a single `Get-Recipient` call could run.

---

## Correct End-to-End Process (Authoritative)

### 1. Connect With Device Code Authentication

`Connect-ExchangeOnline` supports an auth flow that never asks MSAL to launch a local browser at all:

```powershell
Connect-ExchangeOnline -Device
```

### 2. Complete Sign-In in Any Browser

The command prints a URL and a one-time code to the console. Open the URL in *any* browser — the same machine, a different machine, a phone, doesn't matter — enter the code and sign in. The PowerShell session picks up the resulting token once you complete it.

Because the local-browser-launch code path is never invoked, MSAL's OS-version gate never gets a chance to throw (see [Issue 1](#issue-1--connect-exchangeonline-throws-platformnotsupportedexception)).

### 3. Apply the Same Pattern to Other MSAL-Backed Modules

The same pattern applies to other MSAL-backed PowerShell modules with an interactive default (`Connect-MgGraph`, `Connect-AzAccount`) if you hit the same or a similar platform-detection failure — look for their device-code equivalent rather than troubleshooting the local browser or your own environment.

### 4. Retry Plain Interactive Sign-In After a Module Update

This category of failure tends to be temporary — it resolves itself once the module (and the MSAL.NET version it bundles) ships an update recognizing the new OS version. Retry plain interactive `Connect-ExchangeOnline` after a module update rather than defaulting to `-Device` forever.

**If this all happens → connected to Exchange Online, no local browser needed.**

---

## Issues Encountered in This Case

### Issue 1 — `Connect-ExchangeOnline` throws `PlatformNotSupportedException`

**Observed:**

```
PS> Connect-ExchangeOnline
Error Acquiring Token:
System.PlatformNotSupportedException: macOS 26.6.2
   at Microsoft.Identity.Client.Platforms.netstandard.NetCorePlatformProxy.StartDefaultOsBrowserAsync(...)
   at Microsoft.Identity.Client.Platforms.Shared.Desktop.OsBrowser.DefaultOsBrowserWebUi...
   ...
OperationStopped: macOS 26.6.2
```

No browser window ever opens. No sign-in prompt appears. The exception fires before authentication even starts, and the "error" is just the OS version string.

**Root cause:** MSAL.NET's desktop platform proxy has to detect the local OS in order to know how to launch the system's default browser for the interactive auth flow. That detection logic is gated by known/supported version ranges baked into the library at build time. When a machine runs a macOS version newer than that build of MSAL recognizes, the version-detection code throws `PlatformNotSupportedException` instead of falling through to "launch the browser anyway" — the library assumes an unrecognized version means an unsupported platform, not just a newer one.

This is a client-library staleness issue, not a real incompatibility: the browser launch itself would almost certainly work fine, MSAL just refuses to try. It shows up as a hard crash rather than a warning, which makes it easy to misread as a permissions, network, or module-installation problem — none of those are the actual cause.

**Fix:** `Connect-ExchangeOnline -Device` ([step 1](#1-connect-with-device-code-authentication)).

## Lessons Learned

- **A `PlatformNotSupportedException` whose message is your own OS version string is a client-library recognition problem, not a permissions, network, or account issue** — don't go looking in the tenant or your credentials for a fix.
- **Device code auth is a universal fallback for MSAL-backed modules** whenever the local-browser launch path breaks, for this reason or any other (headless machines, remote sessions, sandboxed environments).
- **This category of failure tends to be temporary** — it resolves itself once the module (and the MSAL.NET version it bundles) ships an update recognizing the new OS version. Worth retrying plain interactive `Connect-ExchangeOnline` after a module update rather than defaulting to `-Device` forever.
- **An address that doesn't show up as its own mailbox may be a proxy address on another one** — `Get-Recipient` shows what it actually resolves to.

## Quick Reference Checklist

| If you see… | Do this |
|---|---|
| `PlatformNotSupportedException: macOS <version>` on `Connect-ExchangeOnline` | `Connect-ExchangeOnline -Device` |
| No browser opens, no sign-in prompt | Same cause — use device code auth, don't troubleshoot the browser |
| Same failure on `Connect-MgGraph` / `Connect-AzAccount` | Use that module's device-code equivalent |
| Module updated since | Retry plain interactive `Connect-ExchangeOnline` |
| Address not found as a mailbox in the admin center | `Get-Recipient -Identity <address>` — check if it's an `smtp:` proxy on another mailbox |
