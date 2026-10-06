# Device Code Auth Works, Then the Script Signs In Again Anyway

**Stack:** PowerShell 7, ExchangeOnlineManagement (EXO V3), macOS, Exchange Online mail flow (transport) rules

## TL;DR

On a macOS release newer than the bundled MSAL recognises, interactive `Connect-ExchangeOnline` throws `PlatformNotSupportedException` (see [walkthrough 18](18-exo-connect-macos-browser-launch.md)), and `Connect-ExchangeOnline -Device` is the way round it. That works fine at the prompt — but the moment you run a script that calls `Connect-ExchangeOnline` itself, the script throws the exact same exception and your working session is no help. The obvious workaround — define an empty `Connect-ExchangeOnline` function to shadow it — doesn't work either, because in EXO V3 `Connect-ExchangeOnline` is a *module function*, and the script's own `Import-Module ExchangeOnlineManagement` puts the real one straight back. The fix that does work without touching the script on disk: load it into a scriptblock with its connect/disconnect lines stripped, and run that against your existing session.

## How I got here

The task was replacing a paid third-party email signature service with native Exchange Online mail flow rules, ahead of the service's renewal date. A colleague had written three scripts for it:

- one that creates a transport rule per sender, using `ApplyHtmlDisclaimer` with directory tokens (`%%DisplayName%%`, `%%Title%%`, `%%Email%%`, `%%PhoneNumber%%`) so the signature follows each user's Entra profile;
- one that removes those rules;
- one that adds senders to the `ExceptIfFrom` list of the third-party service's own routing rule, so anyone moved to the native signature doesn't get both.

The plan was to pilot it on a single sender — me — before rolling it out. Every script follows the same shape:

```powershell
Import-Module ExchangeOnlineManagement
Connect-ExchangeOnline -ShowBanner:$false
try {
    # ... Get-EXORecipient, Get/New/Set-TransportRule ...
}
finally {
    Disconnect-ExchangeOnline -Confirm:$false
}
```

Perfectly reasonable for a Windows machine where interactive sign-in works. On this Mac it doesn't.

## The symptom

Device code sign-in at the prompt succeeded and the session was healthy:

```
PS> Get-ConnectionInformation | fl State,TokenStatus
State       : Connected
TokenStatus : Active

PS> Get-TransportRule | select -First 3 Name     # works
PS> Get-EXORecipient me@example.com              # works
```

Then, in the same window:

```
PS> .\New-SignatureRule.ps1 -Sender me@example.com -WhatIf
Error Acquiring Token:
System.PlatformNotSupportedException: macOS 27.0.1
   at Microsoft.Identity.Client.Platforms.netstandard.NetCorePlatformProxy.StartDefaultOsBrowserAsync(...)
OperationStopped: macOS 27.0.1
```

Same exception as walkthrough 18, even though the session the script was launched from was already authenticated.

## Dead end: shadowing the connect function

The first idea was to neutralise the script's connect and disconnect calls by defining empty global functions before running it — PowerShell resolves functions ahead of cmdlets, so this normally works for shadowing a cmdlet:

```powershell
function global:Connect-ExchangeOnline {}
function global:Disconnect-ExchangeOnline {}
.\New-SignatureRule.ps1 -Sender me@example.com -WhatIf     # still throws
```

It made no difference.

## Root cause

In the EXO V3 module, `Connect-ExchangeOnline` isn't a compiled cmdlet — it's a **function** exported by the module. So the override isn't "function beats cmdlet"; it's one function replacing another in the same function table, and whichever was defined last wins. The script's first line, `Import-Module ExchangeOnlineManagement`, re-exports the module's functions and puts the real `Connect-ExchangeOnline` back over the stub. The real one tries the local browser, MSAL throws.

Easy to confirm:

```powershell
Import-Module ExchangeOnlineManagement
(Get-Command Connect-ExchangeOnline).CommandType         # Function
function global:Connect-ExchangeOnline { 'STUB' }
(Get-Command Connect-ExchangeOnline).Module              # (empty) - stub is active
& { Import-Module ExchangeOnlineManagement
    (Get-Command Connect-ExchangeOnline).Module }        # ExchangeOnlineManagement - real one is back
```

The session itself was never the problem; it stayed connected throughout. The script just never got as far as using it.

## The fix: strip the lines in memory

Rather than edit a reviewed, merged script just to run it once, load its text, remove the `Import-Module`, `Connect-` and `Disconnect-` lines, and run the result as a scriptblock. Scriptblocks keep the `param()` block and `[CmdletBinding(SupportsShouldProcess)]`, so parameters and `-WhatIf` behave exactly as they do with the file:

```powershell
function NoConnect($Path) {
  [scriptblock]::Create(((Get-Content $Path -Raw) -replace '(?m)^[ \t]*(Import-Module ExchangeOnlineManagement|Connect-ExchangeOnline -ShowBanner:\$false|Disconnect-ExchangeOnline -Confirm:\$false)[ \t]*\r?$', ''))
}

& (NoConnect .\New-SignatureRule.ps1) -Sender me@example.com -WhatIf
# What if: Performing the operation "Create transport rule" on target "Signature - <Name>".
```

Stripping `Disconnect-ExchangeOnline` matters as much as stripping the connect: left in, the `finally` block would tear down the device-code session after the first script and you'd be signing in again for the second.

The proper long-term fix belongs in the scripts themselves — only connect if `Get-ConnectionInformation` shows no active session, and only disconnect if the script made the connection — so they work the same whether they're run on Windows, on a new Mac, or from a session that's already signed in.

## Migration steps: third-party signature service → transport rules

The order used, piloting on one sender before touching anyone else. Steps 1–5 are done (the pilot); 6–8 are the planned remainder.

1. **Check directory data.** Every signature field comes from Entra, so pull job title and phone for all licensed users first and fix any gaps — a blank attribute becomes a blank line in the signature:
   ```bash
   az rest --method get --url "https://graph.microsoft.com/v1.0/users?\$filter=accountEnabled eq true&\$select=displayName,mail,jobTitle,businessPhones,assignedLicenses&\$top=999" \
     --query "value[?length(assignedLicenses)>\`0\`].[displayName,mail,jobTitle,join(',',businessPhones)]" -o tsv
   ```
   Shared/generic mailboxes often have "description" text in the job title field — decide whether they get a signature, and clean up the title first if so.
2. **Host the images** on a public HTTPS location and confirm each one returns `200` with an image content type:
   ```bash
   for f in logo icons facebook linkedin; do curl -s -o /dev/null -w "$f %{http_code} %{content_type}\n" https://example.com/Images/$f.png; done
   ```
3. **Create the rule for one pilot sender**, dry run first:
   ```powershell
   & (NoConnect .\New-SignatureRule.ps1) -Sender me@example.com -WhatIf
   & (NoConnect .\New-SignatureRule.ps1) -Sender me@example.com
   Get-TransportRule "Signature - <Name>" | fl Name,State,Mode,Priority,From
   ```
4. **Exclude the pilot sender from the old service's routing rule**, or they get both signatures. The script keeps the existing exclusions and adds to them:
   ```powershell
   & (NoConnect .\Set-ServiceRuleSenderExclusion.ps1) -Sender me@example.com -WhatIf
   & (NoConnect .\Set-ServiceRuleSenderExclusion.ps1) -Sender me@example.com
   ```
   Check it in the Exchange admin center: **Mail flow → Rules** → the service's routing rule → exceptions → "sender is".
5. **Wait ~30 minutes, then test** from the pilot mailbox to a personal external address:
   - a new message: exactly one signature, the new one, with all images loading;
   - a reply chain (external reply, then reply again): the signature mustn't stack up;
   - a meeting invite: no signature added.
   If a test shows two signatures or none, the rules haven't propagated yet — wait and resend before troubleshooting. **Mail flow → Message trace** shows which rules a message actually matched.
6. **Roll out to the remaining senders** — the scripts take an array, so it's the same two commands with the full list. *(pending)*
7. **Cut the old service off**: disable its routing rule first (`Disable-TransportRule`, reversible), confirm mail still flows and signs correctly, then remove the rule and its outbound connector. Do this as soon as everyone's on the new rules rather than waiting for the subscription to lapse, or mail keeps being routed to a service that's about to stop processing it. *(pending)*
8. **Cancel the subscription in writing before the notice deadline.** Signature SaaS subscriptions commonly auto-renew annually with a 30-day notice period and no refund for unused time — so give notice early, and use the remaining paid weeks as the cutover window. Get written confirmation back. *(pending)*

Rollback for any sender is the reverse: remove their signature rule and take them back off the exclusion list (`-Remove`).

## Side notes on the transport-rule approach

Things worth knowing before replacing a signature service with `ApplyHtmlDisclaimer` rules, whatever the auth situation:

- **5,000-character limit** on the disclaimer HTML. A full corporate signature with inline styles, a legal footer and a testimonial lands uncomfortably close; inline styles have to be terse because most mail clients ignore `<style>` blocks.
- **Images have to be hosted** on a public HTTPS URL — the rule can't embed them. Check every one returns 200 before the pilot.
- **Repeated signatures in threads.** `Append` puts the signature at the very bottom of the message, below quoted replies, every time. An `ExceptIfSubjectOrBodyContainsWords` on a phrase unique to that sender's signature (e.g. `Email: <address>`) stops it stacking up in a long thread. The phrase has to appear in body text — HTML attributes aren't matched.
- **Exclude calendaring and S/MIME** (`ExceptIfMessageTypeMatches Calendaring`, and a `Content-Type` header match on `multipart/signed` / `pkcs7-mime`) — modifying either breaks it.
- **`ApplyHtmlDisclaimerFallbackAction Ignore`**, otherwise a message the rule can't insert into gets wrapped as an attachment instead of being sent unchanged.
- **Pre-check directory data.** Every token comes from Entra; a missing job title or phone number becomes a blank line in the signature. A Graph query over all licensed users is a quick way to catch it.
- **Rule changes aren't instant** — allow up to ~30 minutes before testing, and use message trace to see which rules a test message actually hit.
- **During a side-by-side migration**, a sender on the new rule must also be excluded from the old service's routing rule, or they get two signatures — and the third-party service can rewrite its own rule if its connection is ever repaired, silently dropping those exclusions.

## Takeaways

- **If a script calls `Connect-ExchangeOnline` itself, your working `-Device` session doesn't help** — the script re-authenticates from scratch and hits the same MSAL failure.
- **You can't shadow EXO V3's `Connect-ExchangeOnline` with a function** when the script imports the module: it's a module function, and `Import-Module` restores it. Check `(Get-Command X).CommandType` before assuming function-over-cmdlet precedence will save you.
- **Loading a script into a scriptblock with a few lines removed** is a clean, no-edit way to run it against an existing session, and keeps `-WhatIf` working.
- **Scripts meant to be run by more than one person should check for an existing connection** rather than unconditionally connecting and disconnecting.
