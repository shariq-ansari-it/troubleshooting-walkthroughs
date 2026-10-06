# Replacing a Third-Party Email Signature Service With Exchange Transport Rules

## Purpose of this Document

A case study and reference runbook for replacing a paid email signature SaaS with native Exchange Online mail flow (transport) rules — one HTML signature per sender, filled in from each user's Entra profile.

It is intentionally written to:

- Give the complete, working end-to-end process so it can be repeated without help — including the scripts
- Keep the correct path separate from the roadblocks hit along the way
- Record each roadblock with its symptom, cause and fix, so it can be recognised quickly next time
- Act as a memory refresh for a task that only comes round once (and then for joiners/leavers)

## Environment

- Exchange Online (Microsoft 365 Business Premium)
- Existing third-party cloud signature service (transport rule + outbound/inbound connectors)
- Microsoft Entra ID — cloud-only users
- Azure Storage static website for image hosting
- Admin workstation: macOS, PowerShell 7, ExchangeOnlineManagement (EXO V3), Azure CLI
- Small organisation — ~20 senders, one standard signature template

---

## Correct End-to-End Process (Authoritative)

This is the clean path to follow for future migrations.

### 1. Inventory the Current Setup

Cloud signature products usually hook into Exchange Online with:

- a **transport rule** (e.g. *"Identify messages to send to <service>"*) that routes outbound mail to…
- an **outbound connector** pointing at the vendor's smart host, which stamps the signature and returns the message via…
- an **inbound connector**.

Some also add an **Outlook add-in** and an **Entra enterprise application**. All of these get removed at the end, so note their names now.

Connect (use `-Device` on macOS — see [Issue 1](#issue-1--connect-exchangeonline-throws-platformnotsupportedexception)):

```powershell
Connect-ExchangeOnline -Device -UserPrincipalName admin@example.com
```

Then:

```powershell
# All transport rules, in priority order
Get-TransportRule | Sort-Object Priority | Format-Table Priority, Name, State, Mode -AutoSize

# The service's routing rule in detail
Get-TransportRule | Where-Object Name -like '*<service>*' |
    Format-List Name, State, Priority, FromScope, SentToScope, From, FromMemberOf,
                ExceptIfFrom, ExceptIfFromMemberOf, RouteMessageOutboundConnector

# Connectors
Get-OutboundConnector | Format-List Name, Enabled, SmartHosts, IsTransportRuleScoped
Get-InboundConnector  | Format-List Name, Enabled, SenderDomains, TlsSenderCertificateName
```

Record:

- routing rule name, and any existing `ExceptIfFrom` entries
- who the rule covers (everyone, or a group — this tells you whether shared mailboxes currently get a signature)
- outbound and inbound connector names

Also check:

- **Microsoft 365 admin center → Settings → Integrated apps** → vendor Outlook add-in
- **Entra admin center → Enterprise applications** → vendor app

### 2. Check Directory Data

Every signature field comes from Entra — a blank attribute becomes a blank line.

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

Check for:

- **Phone format** — what's stored is what prints (`+442071234567` vs `+44 20 7123 4567`)
- **Shared mailboxes** with description text as the job title ("Shared mailbox with archiving") — decide if they get a signature; fix the title first if so
- **Leavers / disabled-but-licensed accounts** — leave out of the rollout list

### 3. Build the Signature HTML

1. Send yourself an email with the current vendor signature and view its source.
2. Rebuild it as a single `<table>` with **inline styles only** (mail clients ignore `<style>` blocks).
3. Replace per-person values with tokens:

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
| `%%Street%%`, `%%City%%`, `%%PostalCode%%`, `%%CountryOrRegion%%` | Address |

4. Hard-code fixed details (address, switchboard, website, legal text).

Must-haves:

- A **visible text line** `Email: %%Email%%` — the rule uses it to detect a signature already in the thread. Text inside an `href` isn't matched.
- `width`, `height` and `alt` on every `<img>`
- **Under 5,000 characters** total — Exchange rejects longer disclaimer text
- A little **top padding** on the outer table — the signature is appended straight after the last line of the body

The template lives in the create script — see [Appendix A](#appendix-a--new-signatureruleps1).

### 4. Host the Images

Images must be on a public HTTPS URL with stable file names.

```bash
# One-off: enable static website on the storage account
az storage blob service-properties update --account-name <storageacct> \
  --static-website --index-document index.html --auth-mode login

# Upload into Images/
az storage blob upload-batch --account-name <storageacct> --auth-mode login \
  -d '$web' --destination-path Images -s ./signature-images --overwrite \
  --content-type image/png --pattern '*.png'
```

Served at `https://<storageacct>.z33.web.core.windows.net/Images/` (zone varies), or a custom domain in front of it.

Verify:

```bash
for f in logo icons facebook linkedin twitter youtube leaf; do
  curl -s -o /dev/null -w "$f %{http_code} %{content_type}\n" https://downloads.example.com/Images/$f.png
done
```

Expected: every line `200 image/png`.

### 5. Put the Scripts in Source Control

Three scripts, kept together (e.g. `Exchange/` in the infrastructure repo) with a copy of the images:

| Script | Purpose |
|---|---|
| [`New-SignatureRule.ps1`](#appendix-a--new-signatureruleps1) | Create/update one rule per sender (`Signature - <Display Name>`) |
| [`Remove-SignatureRule.ps1`](#appendix-b--remove-signatureruleps1) | Remove rules — rollback / leavers |
| [`Set-ServiceRuleSenderExclusion.ps1`](#appendix-c--set-servicerulesenderexclusionps1) | Add/remove senders on the old service's `ExceptIfFrom` list |

All support `-WhatIf` and reuse an existing Exchange Online session.

Commit them **before** running anything.

### 6. Pilot on One Mailbox

Use your own mailbox. **Everything below runs in one PowerShell window** (see [Issue 3](#issue-3--powershell-commands-pasted-into-zsh)).

From zsh:

```bash
cd ~/Repos/<infra-repo> && git checkout main && git pull
pwsh
```

Wait for the `PS >` prompt, then:

```powershell
cd Exchange
Connect-ExchangeOnline -Device -UserPrincipalName me@example.com
```

Open the printed URL, enter the code, sign in. Back at `PS >`:

```powershell
# Signature rule - dry run, then real
.\New-SignatureRule.ps1 -Sender me@example.com -WhatIf
.\New-SignatureRule.ps1 -Sender me@example.com

# Exclude from the old service - dry run, then real
.\Set-ServiceRuleSenderExclusion.ps1 -Sender me@example.com -WhatIf
.\Set-ServiceRuleSenderExclusion.ps1 -Sender me@example.com

# Confirm
Get-TransportRule "Signature - <Your Name>" | Format-List Name, State, Mode, Priority, From
(Get-TransportRule "Identify messages to send to <service>").ExceptIfFrom
```

Expected `-WhatIf` output:

```
What if: Performing the operation "Create transport rule" on target "Signature - <Your Name>".
Current exclusions: <existing entries>
What if: Performing the operation "ExceptIfFrom: add me@example.com" on target "Identify messages to send to <service>".
```

Expected confirm output:

```
Name     : Signature - <Your Name>
State    : Enabled
Mode     : Enforce
Priority : <n>
From     : {me@example.com}
```

Check in **Exchange admin center (admin.exchange.microsoft.com) → Mail flow → Rules**:

- `Signature - <Your Name>` → Enabled
- service routing rule → *Except if* → *sender is* → includes you

Optional safer first run: add `-Mode Audit` — rule evaluates and logs (visible in message trace) without changing mail; rerun without it to enforce.

### 7. Test

**Wait ~30 minutes** — rule changes take time to replicate.

Send from the work mailbox to a personal external address:

| # | Test | How | Expected |
|---|---|---|---|
| 1 | New message | Compose new → personal address | One signature (the new one), all images, correct name/title/email/phone |
| 2 | Inbound | Reply from personal → work | Arrives in work inbox untouched |
| 3 | Reply chain | In **work** mailbox, reply to that reply → personal | No duplicate signature; **no new signature** on this reply (by design) |
| 4 | Meeting invite | Send invite → personal | No signature, `.ics` intact |

Test 3 must be sent **from the work mailbox** — see [Issue 4](#issue-4--reply-chain-test-looked-incomplete).

If a test shows two signatures or none:

- wait another 15–30 min and resend (propagation)
- **Exchange admin center → Mail flow → Message trace** → open the message → shows which rules matched
- confirm the exclusion is still on the routing rule (see [Lessons Learned](#lessons-learned))

### 8. Roll Out

```powershell
$senders = @(
    'first.last@example.com'
    'second.person@example.com'
    # ... people only, unless shared mailboxes were agreed in step 2
)

.\New-SignatureRule.ps1 -Sender $senders -WhatIf
.\New-SignatureRule.ps1 -Sender $senders

.\Set-ServiceRuleSenderExclusion.ps1 -Sender $senders -WhatIf
.\Set-ServiceRuleSenderExclusion.ps1 -Sender $senders

Get-TransportRule | Where-Object Name -like 'Signature - *' |
    Format-Table Name, State, Mode, @{n='From';e={$_.From -join ';'}} -AutoSize
```

Tell users first — the signature looks slightly different and only appears on the first message in a thread.

### 9. Cut the Old Service Off

Do this as soon as everyone is moved — not when the subscription lapses, or mail keeps routing to a service that's about to stop.

```powershell
# . Disable (reversible)
Disable-TransportRule -Identity "Identify messages to send to <service>" -Confirm:$false
```

Test, check message trace, leave it a day. Then:

```powershell
# . Remove rule + connectors (names from step 1)
Remove-TransportRule     -Identity "Identify messages to send to <service>" -Confirm:$false
Remove-OutboundConnector -Identity "<service outbound connector>"           -Confirm:$false
Remove-InboundConnector  -Identity "<service inbound connector>"            -Confirm:$false

# . Confirm nothing left
Get-TransportRule | Where-Object Name -like '*<service>*'
Get-OutboundConnector; Get-InboundConnector
```

Also remove, if present:

- Outlook add-in — **Microsoft 365 admin center → Settings → Integrated apps**
- Enterprise app — **Entra admin center → Enterprise applications** (check its sign-in logs are idle first)

The `ExceptIfFrom` exclusions go with the routing rule — nothing else to clean up.

### 10. Cancel the Subscription

Signature SaaS typically: **annual auto-renew**, **30 days' written notice** before renewal, **no refund** for unused time.

1. Find the renewal date (renewal reminder email / vendor customer portal)
2. Submit cancellation in writing (portal or support ticket) with the subscription ID; ask for written confirmation it won't renew
3. Forward the confirmation to whoever pays the invoices

Notice and cutover are independent — give notice early, use the remaining paid weeks as the cutover window.

**If this all happens → migration complete.**

---

## Ongoing: Joiners, Leavers, Changes

| Event | Action |
|---|---|
| Joiner | Set job title + phone in Entra → `.\New-SignatureRule.ps1 -Sender new.person@example.com` |
| Leaver | `.\Remove-SignatureRule.ps1 -Sender leaver@example.com` (mailbox already gone: `Remove-TransportRule "Signature - <Name>"`) |
| Name / title / phone change | Update Entra only — tokens pick it up |
| Display name change | Rule name is now stale: remove the old rule by name, rerun create |
| Signature design change | Edit HTML in the script, commit, rerun for everyone (existing rules update in place) |

Rerun for everyone:

```powershell
$all = Get-TransportRule | Where-Object Name -like 'Signature - *' | ForEach-Object { $_.From }
.\New-SignatureRule.ps1 -Sender $all -WhatIf
.\New-SignatureRule.ps1 -Sender $all
```

## Admin Center Alternative (Single Rule)

1. **Exchange admin center → Mail flow → Rules → + Add a rule → Apply disclaimers**
2. **Name:** `Signature - <Display Name>`
3. **Apply this rule if:** *The sender* → *is this person* → user
4. **Do the following:** *Append the disclaimer* → **Enter text** → paste HTML → **Select one** → *Ignore*
5. **Except if:**
   - *The message* → *type is* → *Calendaring*
   - *The subject or body* → *subject or body includes any of these words* → `Email: user@example.com`
   - *A message header* → *matches these text patterns* → `Content-Type` → `multipart/signed`, `pkcs7-mime`
6. **Rule mode:** Enforce → Save → **enable** the rule
7. Service routing rule → **Except if** → *The sender* → *is this person* → add the user

Fine for one; use the scripts for more.

---

## Issues Encountered in This Case

### Issue 1 — `Connect-ExchangeOnline` throws `PlatformNotSupportedException`

**Observed:** `PlatformNotSupportedException: macOS <version>` as soon as `Connect-ExchangeOnline` runs; no browser opens.

**Fix:** `Connect-ExchangeOnline -Device`. Root cause and full detail: [walkthrough 18](18-exo-connect-macos-browser-launch.md).

### Issue 2 — Script fails with the same exception after a working `-Device` sign-in

**Observed:** session at the prompt was healthy —

```powershell
Get-ConnectionInformation | fl State,TokenStatus    # Connected / Active
Get-TransportRule | select -First 3 Name            # works
Get-EXORecipient me@example.com                     # works
```

— but running a script in the same window threw the Issue 1 exception again.

**Root cause:** the script unconditionally ran `Import-Module ExchangeOnlineManagement` and `Connect-ExchangeOnline`, ignoring the existing session and starting a fresh browser sign-in.

**What didn't work:** shadowing the connect function —

```powershell
function global:Connect-ExchangeOnline {}; function global:Disconnect-ExchangeOnline {}
```

In EXO V3, `Connect-ExchangeOnline` is a **module function**, not a cmdlet. The script's `Import-Module` re-exports it over the stub:

```powershell
(Get-Command Connect-ExchangeOnline).CommandType   # Function
```

**Fix (no edit to the script):** load it into a scriptblock with the import/connect/disconnect lines stripped, then run that against the existing session:

```powershell
function NoConnect($Path) {
  [scriptblock]::Create(((Get-Content $Path -Raw) -replace '(?m)^[ \t]*(Import-Module ExchangeOnlineManagement|Connect-ExchangeOnline -ShowBanner:\$false|Disconnect-ExchangeOnline -Confirm:\$false)[ \t]*\r?$', ''))
}
& (NoConnect .\Some-Script.ps1) -Sender me@example.com -WhatIf
```

- `param()` and `-WhatIf` still work (scriptblocks keep `[CmdletBinding()]`)
- Stripping `Disconnect` matters too — otherwise the first script's `finally` block ends the session

**Permanent fix:** scripts check for an existing session and only disconnect if they connected. The scripts in the appendices already do this:

```powershell
$connectedHere = $false
if (-not (Get-Command Get-ConnectionInformation -ErrorAction SilentlyContinue) -or
    -not (Get-ConnectionInformation | Where-Object State -eq 'Connected')) {
    Import-Module ExchangeOnlineManagement
    Connect-ExchangeOnline -ShowBanner:$false
    $connectedHere = $true
}
try { ... } finally { if ($connectedHere) { Disconnect-ExchangeOnline -Confirm:$false } }
```

### Issue 3 — PowerShell commands pasted into zsh

**Observed:** pasted the `git pull`, `pwsh`, `Connect-ExchangeOnline` and `function ...` lines as one block; terminal stuck at:

```
function function>
```

**Root cause:** `pwsh` hadn't started yet when the remaining lines arrived, so zsh tried to parse the PowerShell `function` lines itself.

**Fix:** `Ctrl+C`. Run `pwsh` on its own, wait for `PS >`, then paste PowerShell lines. Keep the same window for everything — the device-code session only exists in that PowerShell process (same principle as the Graph session in [walkthrough 07](07-az-login-cant-query-intune.md)).

### Issue 4 — Reply-chain test looked incomplete

**Observed:**

- reply from the personal address didn't appear in the personal mailbox view
- the work mailbox's reply-to-reply arrived with **no new signature**, only the quoted one

**Root cause:**

- a reply from the personal address goes to the **work** inbox — and isn't sent through the rule at all (the rule only matches mail *from* the sender)
- the rule's `ExceptIfSubjectOrBodyContainsWords "Email: <address>"` matched the signature already in the quoted text, so the rule skipped the message — exactly what stops signatures stacking

**Fix:** none needed — working as designed. Test step 3 is the **work → personal** reply. Agree the "signature on first message only" behaviour with the business before rollout.

---

## Pilot Test Results

| Test | Result |
|---|---|
| New message | One signature, all images, Entra fields correct |
| Inbound | Untouched |
| Reply chain | No duplicates; no new signature on later replies (by design) |
| Meeting invite | No signature, `.ics` intact |
| Dark-mode client | Renders; logo on its own white background |

## Lessons Learned

- A signature SaaS doing one standard template is replaceable with native rules + directory tokens
- **One rule per sender**, because the "already signed" check needs a phrase unique to that sender — a shared rule either stacks signatures or suppresses yours when a colleague's is quoted
- **`Append` = very bottom** of the message, after quoted text; users don't see it while composing or in Sent Items
- **Signature on first message only** is the visible behavioural change — agree it up front
- **Internal mail is signed too** unless you add `-SentToScope NotInOrganization`
- **Phone format comes from Entra** — fix it in the directory, not the rule
- **The vendor can rewrite its routing rule** if its connection is ever repaired, silently dropping `ExceptIfFrom` exclusions — first thing to check if double signatures reappear during side-by-side
- `Set-TransportRule -ExceptIfFrom` **replaces** the whole list — always merge with existing entries
- **Scripts should reuse an existing EXO session** — anything that connects unconditionally breaks device-code workflows
- **Notice deadline ≠ cutover deadline** — give notice early

## Quick Reference Checklist

| If you see… | Do this |
|---|---|
| `PlatformNotSupportedException: macOS <version>` | `Connect-ExchangeOnline -Device` |
| Same exception from a script after `-Device` worked | Script reconnects itself → `NoConnect` workaround, or fix script to reuse session |
| `function function>` in terminal | `Ctrl+C`, start `pwsh` first, then paste |
| Two signatures | Sender missing from service rule's `ExceptIfFrom`, or not propagated yet (wait 30 min) |
| No signature on a new message | Rule not propagated / disabled / wrong `From` → message trace |
| No signature on a reply | Expected — sender's signature already in the thread |
| Blank line in signature | Missing Entra attribute (title/phone) |
| `Signature HTML is N characters` error | Over 5,000 — shorten inline styles / text |

---

## Appendix A — `New-SignatureRule.ps1`

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

## Appendix B — `Remove-SignatureRule.ps1`

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

## Appendix C — `Set-ServiceRuleSenderExclusion.ps1`

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
        # -ExceptIfFrom replaces the whole list, so write back the merged list; empty clears it
        $value = if ($updated.Count) { $updated.ToArray() } else { $null }
        Set-TransportRule -Identity $RuleName -ExceptIfFrom $value
        (Get-TransportRule -Identity $RuleName).ExceptIfFrom | ForEach-Object { Write-Host "  Excluded: $_" }
    }
}
finally {
    if ($connectedHere) { Disconnect-ExchangeOnline -Confirm:$false }
}
```

---

*This document is intended as a long-term reference for repeating the migration and for day-to-day joiner/leaver signature changes.*
