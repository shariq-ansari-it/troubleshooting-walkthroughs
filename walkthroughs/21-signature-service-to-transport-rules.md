# Replacing a Third-Party Email Signature Service With Exchange Transport Rules

**Stack:** Exchange Online mail flow (transport) rules, `ApplyHtmlDisclaimer`, Entra ID attributes, Azure Storage static website, PowerShell 7 / ExchangeOnlineManagement (EXO V3)

## TL;DR

A paid signature SaaS was coming up for annual renewal, and everything it was doing for a small organisation — one standard HTML signature, filled in from each user's directory profile — can be done natively with Exchange Online mail flow rules at no extra cost. The approach: one `ApplyHtmlDisclaimer` rule per sender using Entra attribute tokens, images hosted on a public HTTPS URL, senders excluded from the old service's routing rule as they move over, piloted on one mailbox before rollout. It works, with one real behavioural difference worth agreeing up front: the native rule signs the first message in a thread but not later replies.

This write-up is meant to be a complete runbook — everything needed to do it again from scratch, including the scripts.

## Contents

1. [How the old service is wired in](#how-the-old-service-is-wired-in)
2. [Prerequisites](#prerequisites)
3. [Step 1 — Inventory the current setup](#step-1--inventory-the-current-setup)
4. [Step 2 — Check directory data](#step-2--check-directory-data)
5. [Step 3 — Build the signature HTML](#step-3--build-the-signature-html)
6. [Step 4 — Host the images](#step-4--host-the-images)
7. [Step 5 — The scripts](#step-5--the-scripts)
8. [Step 6 — Pilot on one mailbox](#step-6--pilot-on-one-mailbox)
9. [Step 7 — Test](#step-7--test)
10. [Step 8 — Roll out](#step-8--roll-out)
11. [Step 9 — Cut the old service off](#step-9--cut-the-old-service-off)
12. [Step 10 — Cancel the subscription](#step-10--cancel-the-subscription)
13. [Ongoing: joiners, leavers, signature changes](#ongoing-joiners-leavers-signature-changes)
14. [Doing it in the admin center instead](#doing-it-in-the-admin-center-instead)
15. [Pilot test results](#pilot-test-results)
16. [Design notes and gotchas](#design-notes-and-gotchas)
17. [Side note: running EXO scripts from a new macOS release](#side-note-running-exo-scripts-from-a-new-macos-release)

## How the old service is wired in

Cloud signature products typically hook into Exchange Online with three pieces:

- a **transport rule** (e.g. *"Identify messages to send to <service>"*) that matches outbound mail and routes it to…
- an **outbound connector** pointing at the vendor's smart host, which stamps the signature and sends the message back in via…
- an **inbound connector** that accepts the returned mail.

Some also install an **Outlook add-in** and an **Entra enterprise application** (for directory sync). All of these need finding and removing at the end.

The native replacement needs none of that: a transport rule with `ApplyHtmlDisclaimer` stamps the signature inside Exchange itself.

## Prerequisites

- **Role:** Exchange Administrator (or a role group with *Transport Rules* and *Mail Recipients*) to create transport rules.
- **Module:**
  ```powershell
  Install-Module ExchangeOnlineManagement -Scope CurrentUser
  ```
- **Connect** (use `-Device` if the browser sign-in fails — see the side note at the end):
  ```powershell
  Connect-ExchangeOnline -UserPrincipalName admin@example.com
  # or
  Connect-ExchangeOnline -Device -UserPrincipalName admin@example.com
  ```
- **Somewhere public to host images** over HTTPS (this used an Azure Storage static website).
- **The notice date** for the old subscription — check the renewal email or contract before starting (see Step 10).

## Step 1 — Inventory the current setup

Find exactly what the old service has put in place, and note the names — you'll need them later.

```powershell
# All transport rules, in priority order
Get-TransportRule | Sort-Object Priority |
    Format-Table Priority, Name, State, Mode -AutoSize

# The service's routing rule in detail (adjust the name filter)
Get-TransportRule | Where-Object Name -like '*<service>*' |
    Format-List Name, State, Priority, FromScope, SentToScope, From, FromMemberOf,
                ExceptIfFrom, ExceptIfFromMemberOf, RouteMessageOutboundConnector

# Connectors
Get-OutboundConnector | Format-List Name, Enabled, SmartHosts, IsTransportRuleScoped
Get-InboundConnector  | Format-List Name, Enabled, SenderDomains, TlsSenderCertificateName
```

Things to note:

- the **routing rule name** (needed for the exclusion script);
- any existing `ExceptIfFrom` entries (the exclusion script keeps them);
- whether the rule is scoped to everyone or a group (tells you who currently gets a signature — including shared mailboxes);
- the **outbound and inbound connector names** (removed in Step 9).

Also check **Microsoft 365 admin center → Settings → Integrated apps** for an Outlook add-in, and **Entra admin center → Enterprise applications** for the vendor's app.

## Step 2 — Check directory data

Every signature field comes from Entra, so a blank attribute becomes a blank line. Pull job title and phone for all licensed users:

```bash
az rest --method get \
  --url "https://graph.microsoft.com/v1.0/users?\$filter=accountEnabled eq true&\$select=displayName,mail,jobTitle,businessPhones,mobilePhone,assignedLicenses&\$top=999" \
  --query "value[?length(assignedLicenses)>\`0\`].[displayName,mail,jobTitle,join(',',businessPhones)]" -o tsv
```

Fix gaps in **Entra admin center → Users → <user> → Properties → Job information / Contact information**, or:

```bash
az rest --method patch --url "https://graph.microsoft.com/v1.0/users/first.last@example.com" \
  --body '{"jobTitle":"Consultant","businessPhones":["+44 20 7123 4567"]}'
```

Watch for:

- **Phone format** — whatever is stored is what prints. `+442071234567` and `+44 20 7123 4567` look very different in a signature.
- **Shared/generic mailboxes** (support@, sales@, accounts@…) often have *description* text in the job title field ("Shared mailbox with archiving"). Decide whether they get a signature; if yes, set a proper title first.
- **Leavers / disabled-but-licensed accounts** — leave them out of the rollout list.

## Step 3 — Build the signature HTML

### Start from the current signature

Send yourself an email with the current (vendor) signature, open it in a browser-based client or "view source", and copy the HTML structure, colours, fonts and image URLs. Rebuild it as a single `<table>` layout with **inline styles only** — most mail clients ignore `<style>` blocks.

### Tokens

`ApplyHtmlDisclaimer` replaces `%%Attribute%%` tokens with the sender's directory values at send time. The useful ones:

| Token | Entra field |
|---|---|
| `%%DisplayName%%` | Display name |
| `%%FirstName%%` / `%%LastName%%` | Given name / surname |
| `%%Title%%` | Job title |
| `%%Department%%` | Department |
| `%%Company%%` | Company name |
| `%%Email%%` | Primary email address |
| `%%PhoneNumber%%` | Business phone |
| `%%MobileNumber%%` | Mobile phone |
| `%%FaxNumber%%` | Fax |
| `%%Office%%` | Office location |
| `%%Street%%`, `%%City%%`, `%%PostalCode%%`, `%%CountryOrRegion%%` | Address fields |

Fixed details that don't vary per person (company address, switchboard/fax, website, legal text) go straight into the HTML.

### Must-haves

- A **visible text line** `Email: %%Email%%`. The rule uses `Email: <address>` to detect a signature already in the thread. It must be rendered text — a value inside an `href` attribute isn't matched.
- **Width and height on every `<img>`**, and `alt` text — images are often blocked until the recipient allows them.
- **Under 5,000 characters** in total. Exchange rejects longer disclaimer text. Keep style strings short and reuse them through PowerShell variables (see the script).
- A little **top padding** on the outer table — the signature is appended directly after the last line of the body.

## Step 4 — Host the images

Images must be on a public HTTPS URL with stable file names; the rule can't embed them. An Azure Storage static website works well (the `$web` container is public by design).

```bash
# One-off: enable static website on an existing storage account
az storage blob service-properties update --account-name <storageacct> \
  --static-website --index-document index.html --auth-mode login

# Upload the images into an Images/ folder
az storage blob upload-batch --account-name <storageacct> --auth-mode login \
  -d '$web' --destination-path Images -s ./signature-images --overwrite \
  --content-type image/png --pattern '*.png'
```

The site is served at `https://<storageacct>.z33.web.core.windows.net/` (zone varies by region) — or put a custom domain/CDN in front of it, e.g. `https://downloads.example.com/Images`.

Verify every image before going further:

```bash
for f in logo icons facebook linkedin twitter youtube leaf; do
  curl -s -o /dev/null -w "$f %{http_code} %{content_type}\n" https://downloads.example.com/Images/$f.png
done
```

Every line should show `200 image/png`. Keep a copy of the images in the repo next to the scripts.

## Step 5 — The scripts

Three scripts, kept together (e.g. `Exchange/` in an infrastructure repo). All support `-WhatIf`, and all **reuse an existing Exchange Online session** if one is open — so they work after a `Connect-ExchangeOnline -Device` sign-in and don't log you out between runs.

### `New-SignatureRule.ps1` — create or update a rule per sender

```powershell
<#
.SYNOPSIS
    Creates or updates one Exchange Online transport rule per sender that
    appends the standard HTML signature.
.EXAMPLE
    .\New-SignatureRule.ps1 -Sender first.last@example.com -WhatIf
    .\New-SignatureRule.ps1 -Sender a@example.com, b@example.com
#>
[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory)]
    [string[]]$Sender,
    [ValidatePattern('^https://')]
    [string]$ImageBaseUrl = 'https://downloads.example.com/Images',
    [string]$RuleNamePrefix = 'Signature - ',
    [ValidateSet('Enforce', 'Audit', 'AuditAndNotify')]
    [string]$Mode = 'Enforce'
)

$ErrorActionPreference = 'Stop'
$img = $ImageBaseUrl.TrimEnd('/')

# Short reusable inline styles - keeps the HTML under the 5,000 character limit
$f = 'font-family:Arial;'
$t = "${f}font-size:9pt;"

$signatureHtml = @"
<table cellpadding="0" cellspacing="0" style="${f}border-collapse:collapse;width:500px;margin-top:16px;">
<tr><td><table cellpadding="0" cellspacing="0"><tr>
<td style="vertical-align:middle;padding-right:12px;"><img src="$img/logo.png" width="150" height="130" alt="Example Ltd"></td>
<td style="vertical-align:middle;">
<div style="${f}font-size:14pt;font-weight:bold;color:#0675BA;">%%DisplayName%%</div>
<div style="${f}font-size:10pt;color:#0675BA;padding-bottom:8px;">%%Title%%</div>
<div style="${t}padding-bottom:8px;"><b>Example Ltd</b><br>1 Example Street, London, AB1 2CD</div>
<div style="${t}"><b>Email:</b> <a href="mailto:%%Email%%" style="color:#000;text-decoration:none;">%%Email%%</a></div>
<div style="${t}"><b>Tel:</b> %%PhoneNumber%%</div>
<div style="${t}"><b>Web:</b> <a href="https://www.example.com/" style="color:#000;text-decoration:none;">www.example.com</a></div>
</td></tr></table></td></tr>
<tr><td style="padding:10px 0;"><img src="$img/linkedin.png" width="24" height="24" alt="LinkedIn"></td></tr>
<tr><td style="${f}font-size:7pt;color:#999;">Example Ltd is registered in England and Wales under number 01234567.<br>This e-mail and any attachments are confidential...</td></tr>
</table>
"@

if ($signatureHtml.Length -gt 5000) {
    throw "Signature HTML is $($signatureHtml.Length) characters - Exchange allows at most 5000."
}
Write-Verbose "Signature HTML length: $($signatureHtml.Length)"

# Reuse an existing session; only connect/disconnect if this script made the connection
$connectedHere = $false
if (-not (Get-Command Get-ConnectionInformation -ErrorAction SilentlyContinue) -or
    -not (Get-ConnectionInformation | Where-Object State -eq 'Connected')) {
    Import-Module ExchangeOnlineManagement
    Connect-ExchangeOnline -ShowBanner:$false
    $connectedHere = $true
}

try {
    $ruleParams = @{
        ApplyHtmlDisclaimerText            = $signatureHtml
        ApplyHtmlDisclaimerLocation        = 'Append'
        ApplyHtmlDisclaimerFallbackAction  = 'Ignore'        # send unchanged rather than wrap as attachment
        ExceptIfMessageTypeMatches         = 'Calendaring'   # appending HTML can break meeting requests
        ExceptIfHeaderMatchesMessageHeader = 'Content-Type'  # don't modify S/MIME signed/encrypted mail
        ExceptIfHeaderMatchesPatterns      = 'multipart/signed', 'pkcs7-mime'
        Mode                               = $Mode
    }

    foreach ($address in $Sender) {
        $recipient = Get-EXORecipient -Identity $address
        $smtp      = $recipient.PrimarySmtpAddress
        $ruleName  = "$RuleNamePrefix$($recipient.DisplayName)"

        $senderParams = $ruleParams + @{
            From                               = $smtp
            # Skip if this sender's signature is already in the thread
            ExceptIfSubjectOrBodyContainsWords = "Email: $smtp"
        }

        if (Get-TransportRule -Identity $ruleName -ErrorAction SilentlyContinue) {
            if ($PSCmdlet.ShouldProcess($ruleName, 'Update transport rule')) {
                Set-TransportRule -Identity $ruleName @senderParams
                Write-Host "Updated '$ruleName'."
            }
        }
        elseif ($PSCmdlet.ShouldProcess($ruleName, 'Create transport rule')) {
            New-TransportRule -Name $ruleName @senderParams | Out-Null
            Write-Host "Created '$ruleName'."
        }

        Get-TransportRule -Identity $ruleName -ErrorAction SilentlyContinue |
            Format-List Name, State, Mode, Priority, From
    }
}
finally {
    if ($connectedHere) { Disconnect-ExchangeOnline -Confirm:$false }
}
```

Running it again for a sender who already has a rule **updates** the rule — that's how signature changes are rolled out (see Ongoing).

### `Remove-SignatureRule.ps1` — remove rules (rollback / leavers)

```powershell
[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory)]
    [string[]]$Sender,
    [string]$RuleNamePrefix = 'Signature - '
)
$ErrorActionPreference = 'Stop'

$connectedHere = $false
if (-not (Get-Command Get-ConnectionInformation -ErrorAction SilentlyContinue) -or
    -not (Get-ConnectionInformation | Where-Object State -eq 'Connected')) {
    Import-Module ExchangeOnlineManagement
    Connect-ExchangeOnline -ShowBanner:$false
    $connectedHere = $true
}

try {
    foreach ($address in $Sender) {
        $ruleName = "$RuleNamePrefix$((Get-EXORecipient -Identity $address).DisplayName)"
        if (-not (Get-TransportRule -Identity $ruleName -ErrorAction SilentlyContinue)) {
            Write-Host "No rule '$ruleName' - skipping."
            continue
        }
        if ($PSCmdlet.ShouldProcess($ruleName, 'Remove transport rule')) {
            Remove-TransportRule -Identity $ruleName -Confirm:$false
            Write-Host "Removed '$ruleName'."
        }
    }
}
finally {
    if ($connectedHere) { Disconnect-ExchangeOnline -Confirm:$false }
}
```

For a leaver whose mailbox is already gone (so `Get-EXORecipient` fails), remove by name: `Remove-TransportRule "Signature - <Display Name>"`.

### `Set-ServiceRuleSenderExclusion.ps1` — exclude senders from the old service

```powershell
[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory)]
    [string[]]$Sender,
    [string]$RuleName = 'Identify messages to send to <service>',
    [switch]$Remove
)
$ErrorActionPreference = 'Stop'

$connectedHere = $false
if (-not (Get-Command Get-ConnectionInformation -ErrorAction SilentlyContinue) -or
    -not (Get-ConnectionInformation | Where-Object State -eq 'Connected')) {
    Import-Module ExchangeOnlineManagement
    Connect-ExchangeOnline -ShowBanner:$false
    $connectedHere = $true
}

try {
    $rule = Get-TransportRule -Identity $RuleName -ErrorAction SilentlyContinue
    if (-not $rule) { throw "Rule '$RuleName' not found." }

    $updated = [System.Collections.Generic.List[string]]::new()
    @($rule.ExceptIfFrom | Where-Object { $_ }) | ForEach-Object { $updated.Add($_) }
    Write-Host "Current exclusions: $(if ($updated.Count) { $updated -join ', ' } else { '(none)' })"

    $changes = @()
    foreach ($address in $Sender) {
        $smtp  = (Get-EXORecipient -Identity $address).PrimarySmtpAddress
        $match = $updated | Where-Object { $_ -ieq $smtp }
        if ($Remove) {
            if ($match) { $match | ForEach-Object { [void]$updated.Remove($_) }; $changes += "remove $smtp" }
            else        { Write-Host "$smtp is not excluded - skipping." }
        }
        elseif ($match) { Write-Host "$smtp is already excluded - skipping." }
        else            { $updated.Add($smtp); $changes += "add $smtp" }
    }

    if (-not $changes) { Write-Host 'Nothing to change.'; return }

    $action = 'ExceptIfFrom: ' + ($changes -join ', ')
    if ($PSCmdlet.ShouldProcess($RuleName, $action)) {
        $value = if ($updated.Count) { $updated.ToArray() } else { $null }   # empty list clears the exception
        Set-TransportRule -Identity $RuleName -ExceptIfFrom $value
        (Get-TransportRule -Identity $RuleName).ExceptIfFrom | ForEach-Object { Write-Host "  Excluded: $_" }
    }
}
finally {
    if ($connectedHere) { Disconnect-ExchangeOnline -Confirm:$false }
}
```

Note `Set-TransportRule -ExceptIfFrom` **replaces** the whole list — that's why the script reads the existing entries and writes back the merged list rather than passing only the new address.

Commit all three scripts (and the images) to git **before** running them.

## Step 6 — Pilot on one mailbox

Use your own mailbox. In one PowerShell window:

```powershell
Connect-ExchangeOnline -Device -UserPrincipalName me@example.com   # or without -Device
cd <repo>/Exchange

# 1. Signature rule - dry run, then real
.\New-SignatureRule.ps1 -Sender me@example.com -WhatIf
.\New-SignatureRule.ps1 -Sender me@example.com

# 2. Exclude from the old service - dry run, then real
.\Set-ServiceRuleSenderExclusion.ps1 -Sender me@example.com -WhatIf
.\Set-ServiceRuleSenderExclusion.ps1 -Sender me@example.com

# 3. Confirm
Get-TransportRule "Signature - <Your Name>" | Format-List Name, State, Mode, Priority, From
(Get-TransportRule "Identify messages to send to <service>").ExceptIfFrom
```

Expected `-WhatIf` output:

```
What if: Performing the operation "Create transport rule" on target "Signature - <Your Name>".
Current exclusions: (existing entries)
What if: Performing the operation "ExceptIfFrom: add me@example.com" on target "Identify messages to send to <service>".
```

Check it in **Exchange admin center (admin.exchange.microsoft.com) → Mail flow → Rules**:

- `Signature - <Your Name>` — Enabled.
- the service's routing rule → *Except if* → *sender is* → includes your address.

Want a safer first run? Create the rule with `-Mode Audit`: it evaluates and logs matches (visible in message trace) without changing any mail, then rerun with the default `Enforce`.

## Step 7 — Test

**Wait about 30 minutes** — rule changes take time to replicate. Then, from your work mailbox to a personal external address:

1. **New message.** Expect exactly one signature (the new one), all images loading, name/title/email/phone correct.
2. **Reply chain.** Reply from the personal address, then reply again from the work mailbox. Expect *no duplicate* signature. (By design the second reply gets **no new** signature — see design notes.)
3. **Meeting invite.** Expect no signature and the `.ics` attachment intact. (Teams meeting branding in the invite body is from the Teams meeting settings, not the rule.)
4. **Inbound.** Mail arriving from outside is untouched.

If a message shows **two signatures** or **none** when one was expected:

- wait another 15–30 minutes and resend — usually it's propagation;
- **Exchange admin center → Mail flow → Message trace** → find the message → the detail shows which transport rules matched;
- check the exclusion is still on the routing rule (the vendor can rewrite it — see gotchas).

## Step 8 — Roll out

Build the sender list from the Step 2 query (people only, unless you've decided shared mailboxes get one), then run both scripts with the full list — dry run first:

```powershell
$senders = @(
    'first.last@example.com'
    'second.person@example.com'
    # ...
)

.\New-SignatureRule.ps1 -Sender $senders -WhatIf
.\New-SignatureRule.ps1 -Sender $senders

.\Set-ServiceRuleSenderExclusion.ps1 -Sender $senders -WhatIf
.\Set-ServiceRuleSenderExclusion.ps1 -Sender $senders

# Check
Get-TransportRule | Where-Object Name -like 'Signature - *' |
    Format-Table Name, State, Mode, @{n='From';e={$_.From -join ';'}} -AutoSize
```

Tell users before you do this — the signature looks slightly different and appears only on the first message in a thread.

## Step 9 — Cut the old service off

Do this as soon as everyone is on the new rules, not when the subscription lapses — otherwise mail is still routed through a service that's about to stop processing it.

```powershell
# 1. Disable the routing rule (reversible)
Disable-TransportRule -Identity "Identify messages to send to <service>" -Confirm:$false
```

Send a few test messages, check message trace, give it a day. Then remove everything:

```powershell
# 2. Remove the routing rule and connectors (names from Step 1)
Remove-TransportRule     -Identity "Identify messages to send to <service>" -Confirm:$false
Remove-OutboundConnector -Identity "<service outbound connector>"           -Confirm:$false
Remove-InboundConnector  -Identity "<service inbound connector>"            -Confirm:$false

# 3. Confirm nothing left
Get-TransportRule  | Where-Object Name -like '*<service>*'
Get-OutboundConnector; Get-InboundConnector
```

Also remove, if present:

- the vendor's **Outlook add-in** — *Microsoft 365 admin center → Settings → Integrated apps*;
- the vendor's **enterprise application** — *Entra admin center → Enterprise applications* (check sign-in logs first to confirm it's idle).

Once the routing rule is gone, the `ExceptIfFrom` exclusions disappear with it — nothing else to clean up.

## Step 10 — Cancel the subscription

Signature SaaS subscriptions commonly **auto-renew annually**, need **30 days' written notice** before the renewal date, and give **no refund** for unused time. So:

- find the renewal date (renewal reminder email, or the vendor's customer portal);
- submit the cancellation in writing — via the customer portal or a support ticket — well before the notice deadline, quoting the subscription ID and asking for written confirmation that it won't renew;
- forward the confirmation to whoever pays the invoices.

Notice and cutover are independent: give notice early, and use the remaining paid weeks as the cutover window.

## Ongoing: joiners, leavers, signature changes

Add these to the joiner/leaver checklist:

| Event | Action |
|---|---|
| **Joiner** | Set job title and phone in Entra, then `.\New-SignatureRule.ps1 -Sender new.person@example.com` |
| **Leaver** | `.\Remove-SignatureRule.ps1 -Sender leaver@example.com` (or `Remove-TransportRule "Signature - <Name>"` if the mailbox is already gone) |
| **Name / title / phone change** | Just update Entra — tokens pick it up automatically. If the **display name** changes, the rule name is now stale: remove the old rule by name and run the create script again. |
| **Signature design change** | Edit the HTML in the script, commit, then rerun for everyone — existing rules are updated in place: `.\New-SignatureRule.ps1 -Sender (Get-TransportRule \| ? Name -like 'Signature - *' \| % { $_.From }) -WhatIf` |

## Doing it in the admin center instead

For a single one-off rule without the scripts:

1. **Exchange admin center → Mail flow → Rules → + Add a rule → Apply disclaimers**.
2. **Name:** `Signature - <Display Name>`.
3. **Apply this rule if:** *The sender* → *is this person* → pick the user.
4. **Do the following:** *Append the disclaimer* → **Enter text** → paste the signature HTML → **Select one**: *Ignore* (fallback action).
5. **Except if:**
   - *The message* → *type is* → *Calendaring*;
   - *The subject or body* → *subject or body includes any of these words* → `Email: user@example.com`;
   - *A message header* → *matches these text patterns* → header `Content-Type`, patterns `multipart/signed`, `pkcs7-mime`.
6. **Rule mode:** Enforce. Save, then **enable** the rule (new rules can be created disabled).
7. Open the service's routing rule → **Except if** → *The sender* → *is this person* → add the user.

Fine for one person; the scripts are much less error-prone for twenty.

## Pilot test results

| Test | Result |
|---|---|
| New message to an external address | One signature (the new one), all images loading, Entra fields filled in |
| Inbound mail | Untouched — the rule only matches mail *from* the sender |
| Reply chain (out → external reply → reply again) | No duplicate signature; the second outbound reply got **no** new signature because the first one was already quoted |
| Meeting invite | No signature added, `.ics` intact |
| Dark-mode client | Renders fine; logo sits on its own white background |

## Design notes and gotchas

- **Signature on first message only.** The one real behavioural change from the old service. The "already in the thread" exception that prevents stacking also means a reply in an existing thread isn't signed again — the recipient sees the signature in the quoted history instead. Agree this with whoever owns the brand before rollout.
- **Why one rule per sender, not one rule for everyone:** the "already signed" check needs a phrase unique to *that sender* (`Email: <their address>`). A single shared rule would either stack signatures in long threads or suppress yours because a colleague's signature is already quoted.
- **`Append` means the very bottom.** Transport rules can't place the signature under the reply text like a client-side or add-in signature can; it goes after everything, including quoted content. Users don't see it while composing or in Sent Items.
- **5,000-character limit** on the disclaimer HTML — the script fails early if it's over.
- **Body text, not attributes.** `ExceptIfSubjectOrBodyContainsWords` matches rendered text; the `Email: <address>` phrase must be visible.
- **Phone formatting comes from Entra** — fix it in the directory, not the rule.
- **Internal mail is signed too** — the rule has no `SentToScope`. Add `-SentToScope NotInOrganization` to the rule parameters if you only want external mail signed.
- **Rule priority:** new rules go to the bottom of the list. That's fine here — the old routing rule doesn't stop rule processing, and the exclusion keeps it away from migrated senders.
- **Side-by-side period:** the vendor can rewrite its own routing rule if its connection is ever repaired or reconnected, silently dropping your `ExceptIfFrom` exclusions — check them first if double signatures suddenly reappear.

## Side note: running EXO scripts from a new macOS release

On a macOS version newer than the bundled MSAL recognises, `Connect-ExchangeOnline` throws `PlatformNotSupportedException` — see [walkthrough 18](18-exo-connect-macos-browser-launch.md); the fix is `Connect-ExchangeOnline -Device`.

The scripts above check `Get-ConnectionInformation` first, so they just reuse a `-Device` session. But a script that **unconditionally** calls `Import-Module ExchangeOnlineManagement` and `Connect-ExchangeOnline` will ignore your working session, re-authenticate, and hit the same exception. Shadowing it with `function global:Connect-ExchangeOnline {}` doesn't help: in EXO V3 `Connect-ExchangeOnline` is a *module function*, not a cmdlet, and the script's `Import-Module` puts the real one straight back.

To run such a script unmodified, load it into a scriptblock with the import/connect/disconnect lines stripped (stripping `Disconnect` matters too, or the first script's `finally` block ends the session):

```powershell
function NoConnect($Path) {
  [scriptblock]::Create(((Get-Content $Path -Raw) -replace '(?m)^[ \t]*(Import-Module ExchangeOnlineManagement|Connect-ExchangeOnline -ShowBanner:\$false|Disconnect-ExchangeOnline -Confirm:\$false)[ \t]*\r?$', ''))
}
& (NoConnect .\Some-Script.ps1) -Sender me@example.com -WhatIf
```

The scriptblock keeps `param()` and `[CmdletBinding(SupportsShouldProcess)]`, so parameters and `-WhatIf` behave as normal.

## Takeaways

- **A signature SaaS for a single standard template is often replaceable** with native transport rules and directory tokens — the cost is three small scripts and a joiner/leaver step.
- **Inventory first:** routing rule, both connectors, add-in and enterprise app all need removing at the end.
- **Pilot on one mailbox, test four cases** (new mail, inbound, reply chain, meeting invite) before touching anyone else.
- **Agree the reply-signature behaviour up front** — it's the visible difference users will notice.
- **Give the subscription notice early**; notice deadline and cutover deadline are independent.
