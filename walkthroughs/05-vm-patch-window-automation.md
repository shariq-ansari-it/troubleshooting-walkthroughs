# Automating a VM Patch Window With Azure Automation + Resource Graph

**Stack:** Azure Automation Accounts, PowerShell runbooks, Azure Resource Graph, Managed Identity RBAC

## TL;DR

A fleet of Azure VMs had daily auto-shutdown enabled to save cost, but the weekly maintenance/patch window ran overnight — after the VMs had already shut themselves down, so scheduled patching silently never ran. Built a small Azure Automation Account with two runbooks (start-before, stop-after) driven off a system-assigned managed identity and a live Resource Graph query, so any VM enrolled in a maintenance configuration gets picked up automatically without maintaining a hardcoded VM list anywhere.

## The problem

Cost-saving auto-shutdown and scheduled patching don't talk to each other by default. Auto-shutdown was configured tenant-wide at a sensible evening time; the patch/maintenance window for Azure Update Manager was configured separately, overnight, on a schedule nobody had cross-checked against the shutdown time. Net effect: VMs were off when the maintenance window fired, so scheduled updates for a chunk of the fleet just silently didn't happen, week after week, with no error surfaced anywhere — a VM that's off doesn't fail to patch, it simply isn't there to receive the job.

## Design

Two runbooks, one Automation Account:

- **`StartVMsForPatching`** — fires ~15 minutes before the maintenance window opens. Starts every VM that's enrolled in a matching maintenance configuration.
- **`StopVMsAfterPatching`** — fires ~3.5 hours after the window opens (comfortably past its expected duration). Stops those same VMs again, so the cost-saving shutdown policy is honored the rest of the time.

The interesting design decision was *how* the runbooks decide which VMs to touch. The easy version hardcodes a VM list in the script — but that list drifts the moment anyone adds a new VM to the maintenance rotation and forgets to update the automation too. Instead, both runbooks query **Azure Resource Graph** at runtime for VMs currently assigned to a maintenance configuration, so enrollment is driven entirely by whatever's already true in Azure — add a VM to a maintenance config, and it's automatically covered by the next scheduled run with no separate step.

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

## Permissions

The Automation Account runs under a **system-assigned managed identity** rather than a stored credential or service principal secret — nothing to rotate, nothing to leak. It's scoped with `Virtual Machine Contributor` on exactly the resource groups that hold enrolled VMs, and nothing else. `Az.Compute` and the Resource Graph cmdlets came pre-installed on the Automation Account's default module set, so no extra module import/maintenance was needed to get this running.

## What this deliberately doesn't cover

Not every machine in scope is a pure Azure VM — some are on-prem/hybrid machines managed through Azure Arc for compliance/patching visibility, on their own separate maintenance configuration (security-only updates, different cadence). Azure's start/stop APIs don't apply to those at all — you can't remotely power-cycle a physical or hypervisor-hosted machine through the Azure control plane the same way. Those stayed on their own schedule, explicitly out of scope for this automation, rather than trying to force one system to cover a class of machine it fundamentally can't manage.

## Takeaways

- **Two independently-configured schedules (cost automation and patch automation) will eventually collide if nothing checks them against each other** — this is worth auditing for proactively, not just after you notice patches aren't landing.
- **Query for the current desired state at runtime (Resource Graph, tags, dynamic groups) instead of hardcoding a resource list in automation scripts.** The list will drift; the query won't.
- **A VM that's powered off doesn't error when it misses a scheduled job — it just silently isn't there.** Absence-of-failure is not the same as success; worth explicitly verifying maintenance actually ran, not just that nothing complained.
- **Managed identities remove an entire class of credential-rotation problem** for exactly this kind of "small internal automation" use case — there's rarely a good reason to reach for a stored secret instead when the automation only needs to act within its own tenant.
