# Automating a VM Patch Window With Azure Automation + Resource Graph

## TL;DR

A fleet of Azure VMs had daily auto-shutdown enabled to save cost, but the weekly maintenance/patch window ran overnight — after the VMs had already shut themselves down, so scheduled patching silently never ran. Built a small Azure Automation Account with two runbooks (start-before, stop-after) driven off a system-assigned managed identity and a live Resource Graph query, so any VM enrolled in a maintenance configuration gets picked up automatically without maintaining a hardcoded VM list anywhere.

Months later it turned out the automation had **never started a single VM**, while reporting "Completed" every week. Two more silent failures were stacked underneath: the maintenance window's time zone drifted an hour with daylight saving, and the managed identity couldn't see the maintenance assignments, so Resource Graph returned zero rows without an error. Both are fixed below, and the runbooks now fail loudly instead of succeeding at nothing.

## Purpose of this Document

A case study and reference runbook for the problem summarised in the TL;DR above.

It is intentionally written to:

- Give the working design so it can be rebuilt or extended
- Record why each failure was invisible, so the pattern is recognised next time
- Show how to verify patching actually happened, rather than trusting job status
- State clearly what the automation deliberately doesn't cover

## Environment

- Azure VMs with daily auto-shutdown configured tenant-wide at an evening time
- Azure Update Manager maintenance configurations — weekly overnight patch window, VMs spread across several subscriptions
- Azure Automation Account with PowerShell runbooks (`Az` module bundle pre-installed)
- Azure Resource Graph, queried from the runbook via `Invoke-AzRestMethod`
- System-assigned managed identity with RBAC (`Virtual Machine Contributor` + `Reader`)
- Some on-prem/hybrid machines managed via Azure Arc, on their own separate maintenance configuration (out of scope)

---

## Correct End-to-End Process (Authoritative)

### 1. Cross-Check Shutdown Time Against the Patch Window

Compare the auto-shutdown time with the maintenance window schedule. Here the window ran overnight, after VMs had already shut down — see [Issue 1](#issue-1--patching-silently-never-ran).

### 2. Put Both Schedules in the Same Time Zone (UTC)

The maintenance configuration and the Automation schedules each carry their own time zone. Set **both** to UTC.

A Windows time zone name like `GMT Standard Time` is UK local time and moves an hour with daylight saving; `Etc/UTC` on an Automation schedule does not. Mixing them shifts the gap between "start VMs" and "window opens" twice a year — see [Issue 3](#issue-3--window-opened-before-the-vms-were-started).

```bash
# Check every maintenance configuration's window
az graph query -q "resources
  | where type =~ 'microsoft.maintenance/maintenanceconfigurations'
  | project name, resourceGroup,
            tz = tostring(properties.maintenanceWindow.timeZone),
            start = tostring(properties.maintenanceWindow.startDateTime)" -o table
```

To change it, GET the configuration, edit only `properties.maintenanceWindow.timeZone`, diff against the original, then PUT it back:

```bash
URL="https://management.azure.com/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Maintenance/maintenanceConfigurations/<name>?api-version=2023-04-01"
az rest --method get --url "$URL" -o json > cfg.json
jq '{location, tags, properties: (.properties | .maintenanceWindow.timeZone="UTC")}' cfg.json > put.json
diff <(jq -S '{location,tags,properties}' cfg.json) <(jq -S . put.json)   # expect only the timeZone line
az rest --method put --url "$URL" --body @put.json --query properties.maintenanceWindow.timeZone -o tsv
```

Resource Graph can lag several minutes after a change, so confirm with a direct `az rest` GET rather than a graph query.

### 3. Set Each VM's Patch Mode to Customer-Managed Schedules

A VM assigned to a maintenance configuration is only patched by it if its patch orchestration is `AutomaticByPlatform` with `bypassPlatformSafetyChecksOnUserSchedule = true`. A VM left on `AutomaticByOS` still shows up as assigned but doesn't patch on the schedule — see [Issue 5](#issue-5--one-vm-assigned-but-on-the-wrong-patch-mode).

```bash
# Find VMs not set up for customer-managed schedules
az graph query -q "resources
  | where type =~ 'microsoft.compute/virtualmachines'
  | extend ps = properties.osProfile.windowsConfiguration.patchSettings
  | project name, resourceGroup,
            mode = tostring(ps.patchMode),
            bypass = tostring(ps.automaticByPlatformSettings.bypassPlatformSafetyChecksOnUserSchedule)
  | where mode != 'AutomaticByPlatform' or bypass != 'true'" -o table

# Fix one (works while deallocated; set the nested object as JSON, see Issue 5)
az vm update -g <rg> -n <vm> \
  --set osProfile.windowsConfiguration.patchSettings.patchMode=AutomaticByPlatform \
        'osProfile.windowsConfiguration.patchSettings.automaticByPlatformSettings={"bypassPlatformSafetyChecksOnUserSchedule":true}' \
        osProfile.windowsConfiguration.enableAutomaticUpdates=true \
  --query osProfile.windowsConfiguration.patchSettings -o json
```

### 4. Create the Automation Account and Its Identity

One Automation Account running under a **system-assigned managed identity** — no stored credential or service principal secret, so nothing to rotate or leak.

On every resource group that holds enrolled VMs, grant the identity **both**:

| Role | Why |
|---|---|
| `Virtual Machine Contributor` | Start and stop the VMs |
| `Reader` | Read the maintenance configuration assignments. `Virtual Machine Contributor` has no `Microsoft.Maintenance/*/read`, and without it Resource Graph silently returns zero rows — see [Issue 4](#issue-4--the-runbook-never-started-a-single-vm) |

```bash
for scope in /subscriptions/<sub>/resourceGroups/<rg1> /subscriptions/<sub>/resourceGroups/<rg2>; do
  for role in "Virtual Machine Contributor" "Reader"; do
    az role assignment create --assignee-object-id <principalId> \
      --assignee-principal-type ServicePrincipal --role "$role" --scope "$scope" -o none
  done
done
```

The `Az` bundle in the Automation Account's default runtime already has everything the runbooks need — no extra module import.

### 5. Build the Runbooks Around a Live Resource Graph Query — and Make Them Fail Loudly

Both runbooks query Resource Graph at runtime for VMs currently assigned to a maintenance configuration, rather than using a hardcoded VM list that drifts the moment someone adds a VM to the rotation and forgets the automation. Add a VM to a maintenance config and it's covered by the next scheduled run, no separate step.

The query alone is not enough. Each runbook must also:

- **Throw on a non-200 response** from Resource Graph
- **Throw if the query returns zero VMs** — almost always missing RBAC, never a legitimate result for a patching rotation
- **Log the VM count and each VM's prior power state**, so the job output proves what happened
- **Carry on past a single bad VM but fail the job at the end**, naming every VM that couldn't be started or stopped

A terminating error marks the Automation job **Failed**, which is what turns a silent no-op into something visible. Full runbook in [Appendix A](#appendix-a--startvmsforpatching-runbook); the stop runbook is identical apart from the power-state check and `Stop-AzVM -Force`.

### 6. Schedule the Two Runbooks

| Runbook | When (UTC, matching the window) | Does |
|---|---|---|
| `StartVMsForPatching` | ~15 minutes before the maintenance window opens | Starts every VM enrolled in a maintenance configuration |
| `StopVMsAfterPatching` | ~15 minutes after the window closes | Stops those same VMs, so the cost-saving shutdown policy is honoured the rest of the time |

### 7. Verify Maintenance Actually Ran

Don't take a "Completed" job as success — check the maintenance run itself the morning after.

```bash
# Per-VM outcome of the last run
az graph query -q "maintenanceresources
  | where type =~ 'microsoft.maintenance/applyupdates'
  | where todatetime(properties.startDateTime) > ago(2d)
  | extend vm = tostring(split(properties.resourceId, '/')[8])
  | project vm, start = properties.startDateTime, status = tostring(properties.status)
  | order by status asc, vm asc" -o table

# What was installed
az graph query -q "patchinstallationresources
  | where type !has 'softwarepatches'
  | where todatetime(properties.startDateTime) > ago(2d)
  | extend vm = tostring(split(id, '/')[8])
  | project vm, status = tostring(properties.status),
            installed = toint(properties.installedPatchCount),
            failed = toint(properties.failedPatchCount),
            reboot = tostring(properties.rebootStatus)" -o table
```

| Status | Means |
|---|---|
| `UpdateSuccessfullyApplied` | Patched |
| `ShutdownOrUnresponsive` | VM was off when the window opened — check the start runbook output and the time zones |
| `Failed` (Arc machine) | Check the Arc agent status; an `Expired` agent fails every run |

Also open the start runbook's job output: the first line should read `Found N VMs assigned to a maintenance configuration`, with N matching the enrolled fleet.

To test outside the window, start `StartVMsForPatching` manually after hours, check the output, then run `StopVMsAfterPatching`.

**If this all happens → enrolled VMs are on for every patch window, actually patched, and off the rest of the time, with no list to maintain.**

---

## Issues Encountered in This Case

### Issue 1 — Patching silently never ran

**Observed:** scheduled updates for a chunk of the fleet just didn't happen, week after week, with no error surfaced anywhere.

**Root cause:** cost-saving auto-shutdown and the Update Manager maintenance window were configured separately and never cross-checked. VMs were off when the window fired — a VM that's off doesn't fail to patch, it simply isn't there to receive the job.

**Fix:** start/stop runbooks around the window ([steps 4–6](#4-create-the-automation-account-and-its-identity)).

### Issue 2 — Arc-managed machines can't be power-cycled from Azure

**Observed:** not every machine in scope is a pure Azure VM — some are on-prem/hybrid machines managed through Azure Arc for compliance/patching visibility, on their own maintenance configuration (security-only updates, different cadence).

**Root cause:** Azure's start/stop APIs don't apply to them — you can't remotely power-cycle a physical or hypervisor-hosted machine through the Azure control plane the same way.

**Fix:** left on their own schedule, explicitly out of scope, rather than forcing one system to cover a class of machine it fundamentally can't manage.

### Issue 3 — Window opened before the VMs were started

**Observed:** months after go-live, every Azure VM showed `ShutdownOrUnresponsive` on every weekly run. The always-on Arc machines patched fine at the same time, which made the setup look healthy at a glance.

**Root cause:** the maintenance configurations used `GMT Standard Time` (UK local, shifts with daylight saving) while the runbook schedules used `Etc/UTC`. During summer time the window opened at 01:00 UTC, but the start runbook fired at 01:45 UTC — 45 minutes into a window whose VMs had already been marked as off. It would have quietly fixed itself in winter and broken again in spring.

**Fix:** maintenance window time zone set to `UTC` ([step 2](#2-put-both-schedules-in-the-same-time-zone-utc)). The next run confirmed the window opened at 02:00 UTC as intended — and exposed Issue 4.

### Issue 4 — The runbook never started a single VM

**Observed:** with the window now correctly timed, every VM was still `ShutdownOrUnresponsive`. The start job showed **Completed** in about 13 seconds, with no output and no VM start events in the activity log. Every historical job had zero output lines.

**Root cause:** the managed identity only had `Virtual Machine Contributor`, which doesn't include `Microsoft.Maintenance/configurationAssignments/read`. Resource Graph enforces RBAC by **filtering out rows the caller can't read — it doesn't return an error**. The query returned an empty set, the `foreach` looped over nothing, and the runbook exited cleanly. Running the same query as an admin returned every VM, so it looked correct whenever anyone tested it by hand.

**What didn't help:** job status (Completed), the warning stream (only an unrelated "more than one subscription" notice), and testing the query under a human account.

**Fix:** `Reader` added on each enrolled resource group ([step 4](#4-create-the-automation-account-and-its-identity)), and both runbooks hardened to throw on zero results or a failed query ([step 5](#5-build-the-runbooks-around-a-live-resource-graph-query--and-make-them-fail-loudly)).

### Issue 5 — One VM assigned but on the wrong patch mode

**Observed:** one VM was listed as assigned to the maintenance configuration, but its patch mode was `AutomaticByOS`, so the schedule would never have patched it even with everything else fixed.

**Root cause:** maintenance configuration assignment doesn't change or validate the VM's patch orchestration mode.

**What didn't work:** `az vm update --set ...automaticByPlatformSettings.bypassPlatformSafetyChecksOnUserSchedule=true` fails with `Couldn't find 'automaticByPlatformSettings'` when the parent object doesn't exist yet — `--set` can't create intermediate objects from a dotted path.

**Fix:** set the whole object as JSON (`automaticByPlatformSettings={"bypass...":true}`) — see [step 3](#3-set-each-vms-patch-mode-to-customer-managed-schedules).

## Results

- Time zone fix confirmed: the following weekly run opened at 02:00 UTC, after the start runbook.
- `Reader` assignments confirmed on all enrolled resource groups; hardened runbooks published and verified against source control.
- Full end-to-end confirmation (VMs started by the runbook and patched in the window) pending the next scheduled run.

## Lessons Learned

- **Two independently-configured schedules (cost automation and patch automation) will eventually collide if nothing checks them against each other** — this is worth auditing for proactively, not just after you notice patches aren't landing.
- **Check that the schedules agree on time zone, not just on time.** A Windows time zone name and `Etc/UTC` agree for half the year, so the bug only shows up across a daylight-saving change.
- **Resource Graph returns fewer rows, not an error, when the caller lacks read access.** Any automation driven by a Resource Graph query must treat "zero results" as a failure unless zero is a genuinely valid answer — and the identity needs read on the resource types it queries, not just write on the ones it acts on.
- **Test automation queries as the automation's identity, not your own.** An admin account sees everything, so the query looks correct when you run it by hand.
- **A "Completed" job status means the script didn't throw, not that it did anything.** Make runbooks log what they acted on and throw when they acted on nothing.
- **Query for the current desired state at runtime (Resource Graph, tags, dynamic groups) instead of hardcoding a resource list in automation scripts.** The list will drift; the query won't.
- **A VM that's powered off doesn't error when it misses a scheduled job — it just silently isn't there.** Absence-of-failure is not the same as success; verify the maintenance run results ([step 7](#7-verify-maintenance-actually-ran)), not the automation's job status.
- **Managed identities remove an entire class of credential-rotation problem** for exactly this kind of "small internal automation" use case — there's rarely a good reason to reach for a stored secret instead when the automation only needs to act within its own tenant.

## Quick Reference Checklist

| If you see… | Do this |
|---|---|
| Patches not landing, no errors anywhere | Check auto-shutdown time against the maintenance window |
| Every Azure VM `ShutdownOrUnresponsive`, Arc machines fine | Compare the maintenance window time zone with the runbook schedule time zone; set both to UTC |
| Start runbook "Completed" in seconds with no output | Identity can't read maintenance assignments — add `Reader` on the enrolled resource groups |
| Start runbook Failed: "returned 0 VMs" | Same as above — the hardened runbook caught it |
| VM assigned to the config but never patched | Check patch mode is `AutomaticByPlatform` with the bypass flag `true` |
| `az vm update --set` says it can't find `automaticByPlatformSettings` | Set the whole object as JSON instead of a dotted path |
| Arc machine `Failed` every run | Check the Arc agent status — re-onboard if `Expired` |
| Arc machines stuck on "Reboot Required" | The configuration uses `NeverReboot`; schedule a restart separately |
| New VM added to a maintenance config | Set its patch mode (step 3); the Resource Graph query picks it up next run |
| Automation needs credentials | System-assigned managed identity, `Virtual Machine Contributor` + `Reader` on enrolled RGs only |
| Arc-managed machine in the same patch scope | Out of scope for start/stop — leave on its own maintenance config |

---

## Appendix A — StartVMsForPatching Runbook

PowerShell runbook, Az bundle runtime. Replace the subscription IDs with the subscriptions holding enrolled VMs.

```powershell
$ErrorActionPreference = 'Stop'

Connect-AzAccount -Identity | Out-Null

$subscriptions = @(
    '<subscription-id-1>',
    '<subscription-id-2>'
)

$body = @{
    query         = "maintenanceresources | where type == 'microsoft.maintenance/configurationassignments' | where tolower(tostring(properties.resourceId)) contains 'microsoft.compute/virtualmachines' | project resourceId = tolower(tostring(properties.resourceId))"
    subscriptions = $subscriptions
} | ConvertTo-Json

$response = Invoke-AzRestMethod -Method POST `
    -Uri 'https://management.azure.com/providers/Microsoft.ResourceGraph/resources?api-version=2021-03-01' `
    -Payload $body

if ($response.StatusCode -ne 200) {
    throw "Resource Graph query failed (HTTP $($response.StatusCode)): $($response.Content)"
}

$vmIds = @(($response.Content | ConvertFrom-Json).data | ForEach-Object { $_.resourceId } | Sort-Object -Unique)

# Resource Graph silently drops rows the identity can't read, so zero results usually means missing Reader RBAC
if ($vmIds.Count -eq 0) {
    throw "Resource Graph returned 0 VMs with a maintenance configuration assignment. Check the managed identity has Reader on each patching resource group."
}

Write-Output "Found $($vmIds.Count) VMs assigned to a maintenance configuration"

$failed = @()

foreach ($vmId in $vmIds) {
    $parts = $vmId -split '/'
    $subId = $parts[2]
    $rg    = $parts[4]
    $name  = $parts[8]

    try {
        Set-AzContext -SubscriptionId $subId | Out-Null

        $status     = Get-AzVM -ResourceGroupName $rg -Name $name -Status
        $powerState = ($status.Statuses | Where-Object { $_.Code -like 'PowerState/*' }).Code

        if ($powerState -ne 'PowerState/running') {
            Write-Output "Starting: $name ($rg) [was $powerState]"
            Start-AzVM -ResourceGroupName $rg -Name $name -NoWait | Out-Null
        } else {
            Write-Output "Already running: $name ($rg)"
        }
    } catch {
        Write-Error "Failed: $name ($rg) - $($_.Exception.Message)" -ErrorAction Continue
        $failed += $name
    }
}

if ($failed.Count -gt 0) {
    throw "$($failed.Count) of $($vmIds.Count) VMs failed to start: $($failed -join ', ')"
}

Write-Output "Done: start requested for all $($vmIds.Count) VMs"
```

For `StopVMsAfterPatching`, change the power-state check to `if ($powerState -notin @('PowerState/deallocated', 'PowerState/stopped'))`, call `Stop-AzVM -ResourceGroupName $rg -Name $name -Force -NoWait`, and adjust the messages.
