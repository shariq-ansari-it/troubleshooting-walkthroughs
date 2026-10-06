# Automating a VM Patch Window With Azure Automation + Resource Graph

## TL;DR

A fleet of Azure VMs had daily auto-shutdown enabled to save cost, but the weekly maintenance/patch window ran overnight — after the VMs had already shut themselves down, so scheduled patching silently never ran. Built a small Azure Automation Account with two runbooks (start-before, stop-after) driven off a system-assigned managed identity and a live Resource Graph query, so any VM enrolled in a maintenance configuration gets picked up automatically without maintaining a hardcoded VM list anywhere.

## Purpose of this Document

A case study and reference runbook for the problem summarised in the TL;DR above.

It is intentionally written to:

- Give the working design so it can be rebuilt or extended
- Record why the problem was invisible, so the pattern is recognised next time
- State clearly what the automation deliberately doesn't cover

## Environment

- Azure VMs with daily auto-shutdown configured tenant-wide at an evening time
- Azure Update Manager maintenance configurations — weekly overnight patch window
- Azure Automation Account with PowerShell runbooks
- Azure Resource Graph (`Search-AzGraph`), `Az.Compute` — both in the Automation Account's default module set
- System-assigned managed identity with RBAC (`Virtual Machine Contributor`)
- Some on-prem/hybrid machines managed via Azure Arc, on their own separate maintenance configuration (out of scope)

---

## Correct End-to-End Process (Authoritative)

### 1. Cross-Check Shutdown Time Against the Patch Window

Compare the auto-shutdown time with the maintenance window schedule. Here the window ran overnight, after VMs had already shut down — see [Issue 1](#issue-1--patching-silently-never-ran).

### 2. Create the Automation Account and Its Identity

One Automation Account running under a **system-assigned managed identity** — no stored credential or service principal secret, so nothing to rotate or leak.

Grant it `Virtual Machine Contributor` on exactly the resource groups that hold enrolled VMs, and nothing else.

`Az.Compute` and the Resource Graph cmdlets come pre-installed in the default module set — no extra module import/maintenance needed.

### 3. Build the Runbooks Around a Live Resource Graph Query

Both runbooks query Resource Graph at runtime for VMs currently assigned to a maintenance configuration, rather than using a hardcoded VM list that drifts the moment someone adds a VM to the rotation and forgets the automation. Add a VM to a maintenance config and it's covered by the next scheduled run, no separate step.

```powershell
# Simplified shape of the dynamic enrollment query used in both runbooks
$vms = Search-AzGraph -Query @"
    maintenanceresources
    | where type == 'microsoft.maintenance/configurationassignments'
    | join kind=inner (
        resources
        | where type == 'microsoft.compute/virtualmachines'
      ) on `$left.id == `$right.id
    | project vmId = id, vmName = name, resourceGroup
"@

foreach ($vm in $vms) {
    Start-AzVM -ResourceGroupName $vm.resourceGroup -Name $vm.vmName -NoWait
}
```

### 4. Schedule the Two Runbooks

| Runbook | When | Does |
|---|---|---|
| `StartVMsForPatching` | ~15 minutes before the maintenance window opens | Starts every VM enrolled in a matching maintenance configuration |
| `StopVMsAfterPatching` | ~3.5 hours after the window opens (comfortably past its expected duration) | Stops those same VMs, so the cost-saving shutdown policy is honoured the rest of the time |

### 5. Verify Maintenance Actually Ran

Don't take absence of errors as success — explicitly verify the maintenance ran.

**If this all happens → enrolled VMs are on for every patch window and off the rest of the time, with no list to maintain.**

---

## Issues Encountered in This Case

### Issue 1 — Patching silently never ran

**Observed:** scheduled updates for a chunk of the fleet just didn't happen, week after week, with no error surfaced anywhere.

**Root cause:** cost-saving auto-shutdown and the Update Manager maintenance window were configured separately and never cross-checked. VMs were off when the window fired — a VM that's off doesn't fail to patch, it simply isn't there to receive the job.

**Fix:** start/stop runbooks around the window ([steps 2–4](#2-create-the-automation-account-and-its-identity)).

### Issue 2 — Arc-managed machines can't be power-cycled from Azure

**Observed:** not every machine in scope is a pure Azure VM — some are on-prem/hybrid machines managed through Azure Arc for compliance/patching visibility, on their own maintenance configuration (security-only updates, different cadence).

**Root cause:** Azure's start/stop APIs don't apply to them — you can't remotely power-cycle a physical or hypervisor-hosted machine through the Azure control plane the same way.

**Fix:** left on their own schedule, explicitly out of scope, rather than forcing one system to cover a class of machine it fundamentally can't manage.

## Lessons Learned

- **Two independently-configured schedules (cost automation and patch automation) will eventually collide if nothing checks them against each other** — this is worth auditing for proactively, not just after you notice patches aren't landing.
- **Query for the current desired state at runtime (Resource Graph, tags, dynamic groups) instead of hardcoding a resource list in automation scripts.** The list will drift; the query won't.
- **A VM that's powered off doesn't error when it misses a scheduled job — it just silently isn't there.** Absence-of-failure is not the same as success; worth explicitly verifying maintenance actually ran, not just that nothing complained.
- **Managed identities remove an entire class of credential-rotation problem** for exactly this kind of "small internal automation" use case — there's rarely a good reason to reach for a stored secret instead when the automation only needs to act within its own tenant.

## Quick Reference Checklist

| If you see… | Do this |
|---|---|
| Patches not landing, no errors anywhere | Check auto-shutdown time against the maintenance window |
| New VM added to a maintenance config | Nothing — the Resource Graph query picks it up next run |
| Automation needs credentials | System-assigned managed identity, `Virtual Machine Contributor` on enrolled RGs only |
| Arc-managed machine in the same patch scope | Out of scope for start/stop — leave on its own maintenance config |
