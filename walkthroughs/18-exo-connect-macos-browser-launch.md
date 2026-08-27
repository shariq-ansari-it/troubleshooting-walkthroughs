# `Connect-ExchangeOnline` Throws `PlatformNotSupportedException` on a New macOS Release

**Stack:** PowerShell 7, ExchangeOnlineManagement (EXO V3), macOS, MSAL.NET

## TL;DR

`Connect-ExchangeOnline` (interactive, browser-based auth) can fail immediately with a `PlatformNotSupportedException` on a recently released macOS version — before any login prompt appears, and with no network or account problem involved. The exception message is literally the macOS version string. This is MSAL's cross-platform browser-launch code failing to recognize how new the OS is, not anything wrong with your credentials, your tenant, or your machine. The fix is to skip the local-browser launch path entirely with device code authentication.

## How I got here

The task that led to this was a routine one: move a secondary email address from one shared mailbox to another, so mail sent to that address starts landing in a different inbox. The catch is that an address like that isn't necessarily its own object — it can just as easily be a proxy address (an alias) hanging off a completely unrelated mailbox, and there's no way to tell which from the address alone. Searching for it in the admin center as if it were its own mailbox turns up nothing, which reads like the address doesn't exist at all.

The only reliable way to resolve this is to query Exchange Online directly and see what recipient the address actually resolves to:

```powershell
Get-Recipient -Identity thataddress@example.com | Select-Object DisplayName, RecipientType, RecipientTypeDetails, PrimarySmtpAddress, EmailAddresses
```

That's what actually confirmed the address in question was a secondary `smtp:` proxy address on a different mailbox entirely, not a mailbox of its own — which is also what made the fix safe: removing it from one mailbox's `EmailAddresses` collection and adding it to the other's via `Set-Mailbox`, rather than anything involving forwarding rules or a real second mailbox object.

Getting to that answer meant an interactive Exchange Online PowerShell session in the first place — and that's where the actual gotcha in this write-up showed up, before a single `Get-Recipient` call could even run.

## The symptom

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

## Root cause

MSAL.NET's desktop platform proxy has to detect the local OS in order to know how to launch the system's default browser for the interactive auth flow. That detection logic is gated by known/supported version ranges baked into the library at build time. When a machine is running a macOS version newer than what that build of MSAL recognizes, the version-detection code throws `PlatformNotSupportedException` instead of falling through to "launch the browser anyway" — the library assumes an unrecognized version means an unsupported platform, not just a newer one.

This is a client-library staleness issue, not a real incompatibility: the browser launch itself would almost certainly work fine, MSAL just refuses to try. It shows up as a hard crash rather than a warning, which makes it easy to misread as a permissions, network, or module-installation problem — none of those are the actual cause.

## The fix: device code auth instead of the local browser

`Connect-ExchangeOnline` supports an auth flow that never asks MSAL to launch a local browser at all — device code authentication:

```powershell
Connect-ExchangeOnline -Device
```

This prints a URL and a one-time code to the console. Open the URL in *any* browser — the same machine, a different machine, a phone, doesn't matter — enter the code, sign in, and the PowerShell session picks up the resulting token once you complete it. Because the local-browser-launch code path is never invoked, MSAL's OS-version gate never gets a chance to throw.

Same pattern applies to other MSAL-backed PowerShell modules with an interactive default (`Connect-MgGraph`, `Connect-AzAccount`) if you hit the same or a similar platform-detection failure — look for their device-code equivalent rather than troubleshooting the local browser or your own environment.

## Takeaways

- **A `PlatformNotSupportedException` whose message is your own OS version string is a client-library recognition problem, not a permissions, network, or account issue** — don't go looking in the tenant or your credentials for a fix.
- **Device code auth is a universal fallback for MSAL-backed modules** whenever the local-browser launch path breaks, for this reason or any other (headless machines, remote sessions, sandboxed environments).
- **This category of failure tends to be temporary** — it resolves itself once the module (and the MSAL.NET version it bundles) ships an update recognizing the new OS version. Worth retrying plain interactive `Connect-ExchangeOnline` after a module update rather than defaulting to `-Device` forever.
