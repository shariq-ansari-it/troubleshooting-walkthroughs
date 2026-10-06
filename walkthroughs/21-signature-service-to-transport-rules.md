# Replacing a Third-Party Email Signature Service With Exchange Transport Rules

**Stack:** Exchange Online mail flow (transport) rules, `ApplyHtmlDisclaimer`, Entra ID attributes, PowerShell 7 / ExchangeOnlineManagement (EXO V3)

## TL;DR

A paid signature SaaS was coming up for annual renewal, and everything it was doing for a small organisation — one standard HTML signature, filled in from each user's directory profile — can be done natively with Exchange Online mail flow rules at no extra cost. The approach: one `ApplyHtmlDisclaimer` rule per sender using Entra attribute tokens, images hosted on a public HTTPS URL, senders excluded from the old service's routing rule as they move over, piloted on one mailbox before rollout. It works, with one real behavioural difference worth agreeing up front: the native rule signs the first message in a thread but not later replies.

## Background

The old service worked the usual way for cloud signature products: an Exchange transport rule ("identify messages to send to <service>") routes outbound mail through an outbound connector to the vendor, which stamps the signature and hands the message back. The subscription auto-renewed annually with a 30-day written notice period and no refund for unused time.

The replacement is three small scripts:

- **create** — one transport rule per sender (`Signature - <Display Name>`), taking an array of addresses;
- **remove** — deletes those rules, for rollback or leavers;
- **exclude** — adds senders to the `ExceptIfFrom` list of the old service's routing rule, keeping existing exclusions (with a `-Remove` switch to reverse it).

All three support `-WhatIf`.

## How the rule is built

```powershell
$ruleParams = @{
    From                               = $smtp
    ApplyHtmlDisclaimerText            = $signatureHtml
    ApplyHtmlDisclaimerLocation        = 'Append'
    ApplyHtmlDisclaimerFallbackAction  = 'Ignore'              # send unchanged if it can't be inserted
    ExceptIfMessageTypeMatches         = 'Calendaring'         # don't touch meeting requests/responses
    ExceptIfHeaderMatchesMessageHeader = 'Content-Type'
    ExceptIfHeaderMatchesPatterns      = 'multipart/signed', 'pkcs7-mime'   # don't break S/MIME
    ExceptIfSubjectOrBodyContainsWords = "Email: $smtp"        # this sender's signature is already in the thread
}
New-TransportRule -Name "Signature - $displayName" @ruleParams
```

The signature HTML uses Entra tokens — `%%DisplayName%%`, `%%Title%%`, `%%Email%%`, `%%PhoneNumber%%` — so name, job title and phone follow each user's directory profile with no per-user editing.

Why one rule per sender rather than one rule for everyone: the "already signed" check needs a phrase unique to *that sender's* signature (`Email: <their address>`). A single shared rule would either stack signatures in long threads or suppress yours because a colleague's signature is already quoted.

## Migration steps

Piloted on one sender first; steps 1–5 are the pilot, 6–8 the rollout.

1. **Check directory data.** Every signature field comes from Entra, so pull job title and phone for all licensed users and fix gaps first — a blank attribute becomes a blank line:
   ```bash
   az rest --method get --url "https://graph.microsoft.com/v1.0/users?\$filter=accountEnabled eq true&\$select=displayName,mail,jobTitle,businessPhones,assignedLicenses&\$top=999" \
     --query "value[?length(assignedLicenses)>\`0\`].[displayName,mail,jobTitle,join(',',businessPhones)]" -o tsv
   ```
   Shared/generic mailboxes often have description text in the job title field — decide whether they get a signature at all, and clean up the title first if so.
2. **Host the images** on a public HTTPS location and confirm each returns `200` with an image content type:
   ```bash
   for f in logo icons facebook linkedin; do curl -s -o /dev/null -w "$f %{http_code} %{content_type}\n" https://example.com/Images/$f.png; done
   ```
3. **Create the rule for one pilot sender**, dry run first:
   ```powershell
   .\New-SignatureRule.ps1 -Sender me@example.com -WhatIf
   .\New-SignatureRule.ps1 -Sender me@example.com
   Get-TransportRule "Signature - <Name>" | fl Name,State,Mode,Priority,From
   ```
4. **Exclude the pilot sender from the old service's routing rule**, or they get both signatures:
   ```powershell
   .\Set-ServiceRuleSenderExclusion.ps1 -Sender me@example.com -WhatIf
   .\Set-ServiceRuleSenderExclusion.ps1 -Sender me@example.com
   ```
   Check in the Exchange admin center: **Mail flow → Rules** → the service's routing rule → exceptions → "sender is".
5. **Wait ~30 minutes, then test** from the pilot mailbox to a personal external address (results below). If a test shows two signatures or none, the rules haven't propagated yet — wait and resend before troubleshooting. **Mail flow → Message trace** shows which rules a message actually matched.
6. **Roll out to the remaining senders** — same two scripts with the full address list.
7. **Cut the old service off**: disable its routing rule first (`Disable-TransportRule`, reversible), confirm mail still flows and signs correctly, then remove the rule and its outbound connector. Do this as soon as everyone's moved rather than waiting for the subscription to lapse, or mail keeps being routed to a service that's about to stop processing it.
8. **Cancel the subscription in writing before the notice deadline**, and get written confirmation. Giving notice early costs nothing — the remaining paid weeks are the cutover window.

Rollback for any sender is the reverse: remove their signature rule and take them back off the exclusion list.

## Pilot test results

| Test | Result |
|---|---|
| New message to an external address | One signature (the new one), all images loading, Entra fields filled in |
| Inbound mail | Untouched — the rule only matches mail *from* the sender |
| Reply chain (out → external reply → reply again) | No duplicate signature; the second outbound reply got **no** new signature because the first one was already quoted |
| Meeting invite | No signature added, `.ics` intact |
| Dark-mode client | Renders fine; logo sits on its own white background |

## Design notes and gotchas

- **Signature on first message only.** This is the one real behavioural change from the old service. The "already in the thread" exception that prevents stacking also means a reply in an existing thread isn't signed again — the recipient sees the signature in the quoted history instead. Agree this with whoever owns the brand before rollout.
- **`Append` means the very bottom.** Transport rules can't place the signature under the reply text the way a client-side or add-in signature can; it always goes after everything, including quoted content. Users also don't see it while composing or in Sent Items.
- **5,000-character limit** on the disclaimer HTML. A full corporate signature with inline styles, legal footer and a testimonial gets close; keep inline styles terse (most clients ignore `<style>` blocks) and have the script fail if the HTML goes over.
- **Body text, not attributes.** `ExceptIfSubjectOrBodyContainsWords` matches rendered text — the phrase has to be visible text in the signature, not inside an `href`.
- **Phone formatting comes from Entra.** If `businessPhones` is stored as `+442071234567`, that's what prints; fix it in the directory, not the rule.
- **No spacing above the signature.** It starts immediately after the last line of the body; a little top padding on the outer table helps.
- **New starters and leavers** now need a rule created/removed — add it to the joiner/leaver checklist.
- **Side-by-side period:** the old service can rewrite its own routing rule if its connection is ever repaired, silently dropping your `ExceptIfFrom` exclusions — check them if double signatures suddenly reappear.

## Side note: running the scripts from a new macOS release

On a macOS version newer than the bundled MSAL recognises, `Connect-ExchangeOnline` throws `PlatformNotSupportedException` — see [walkthrough 18](18-exo-connect-macos-browser-launch.md); the fix there is `Connect-ExchangeOnline -Device`.

The twist here: the scripts call `Import-Module ExchangeOnlineManagement` and `Connect-ExchangeOnline` themselves, so a working `-Device` session at the prompt doesn't help — each script re-authenticates and hits the same exception. Shadowing it with `function global:Connect-ExchangeOnline {}` doesn't work either: in EXO V3 `Connect-ExchangeOnline` is a *module function*, not a cmdlet, and the script's `Import-Module` puts the real one straight back.

What worked without editing the scripts: connect with `-Device`, then load each script into a scriptblock with its import/connect/disconnect lines stripped (stripping `Disconnect` matters too, or the first script's `finally` block kills the session):

```powershell
function NoConnect($Path) {
  [scriptblock]::Create(((Get-Content $Path -Raw) -replace '(?m)^[ \t]*(Import-Module ExchangeOnlineManagement|Connect-ExchangeOnline -ShowBanner:\$false|Disconnect-ExchangeOnline -Confirm:\$false)[ \t]*\r?$', ''))
}
& (NoConnect .\New-SignatureRule.ps1) -Sender me@example.com -WhatIf
```

The scriptblock keeps `param()` and `[CmdletBinding(SupportsShouldProcess)]`, so parameters and `-WhatIf` behave as normal. The proper fix belongs in the scripts: connect only if `Get-ConnectionInformation` shows no active session, and only disconnect if the script made the connection.

## Takeaways

- **A signature SaaS for a single standard template is often replaceable** with native transport rules and directory tokens — the cost is a small set of scripts and a joiner/leaver step.
- **Pilot on one mailbox, test four cases** (new mail, inbound, reply chain, meeting invite) before touching anyone else.
- **Agree the reply-signature behaviour up front** — it's the visible difference users and management will notice.
- **Give the subscription notice early**; the notice deadline and the cutover deadline are independent, and the remaining paid period is free cutover time.
