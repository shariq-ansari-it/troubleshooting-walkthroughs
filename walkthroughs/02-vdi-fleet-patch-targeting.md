# Patching a VDI Fleet Without Touching Everyone Else's Session

**Stack:** Microsoft Intune, Microsoft Graph API, Win32 LOB apps, Azure Virtual Desktop

## TL;DR

A softphone/VDI client app started throwing "Update required — your desktop app is no longer supported" on one virtual desktop, while identical VMs in the same Intune group kept working fine. The app vendor enforces a minimum client version server-side; once a version falls below the cutoff, it stops working entirely rather than degrading gracefully. The fix needed a Win32 app supersedence chain in Intune plus a way to push the update to exactly one device without triggering an update — and a possible outage — across the whole fleet at once. Intune Assignment Filters solved the targeting problem; a Graph API quirk almost derailed the supersedence setup.

## Background

The environment: a fleet of Azure Virtual Desktop VMs, each running a VDI-aware softphone client. The vendor ships the client as two logical components — a main app that runs on the VM, and a local plugin that runs on the user's physical endpoint to offload audio/video processing back to their own device (without it, calls still work, but quality suffers and the VM absorbs load it shouldn't have to). Both are deployed as Win32 line-of-business apps through Intune.

The vendor enforces a hard minimum supported version. Below that version, the app doesn't just nag you to update — it refuses to function. One VM in the fleet hit that wall.

## Why not just push the update to the whole group?

Because the rest of the group was working fine, and a same-day fleet-wide app push means a same-day risk of breaking working sessions for everyone else, with no ability to stage or roll back gradually. The goal was: patch the one broken VM immediately, then plan the fleet-wide rollout separately and deliberately.

## Setting up the update itself

Intune's mechanism for "this new app version replaces that old one" is **supersedence**, configured on Win32 LOB apps. The Graph API documentation describes updating relationships via a `relationships` navigation property with an OData type cast on the app object. That path did not work in practice — it silently failed to persist the supersedence relationship.

What actually works is calling the dedicated action endpoint directly, with no cast:

```
POST /beta/deviceAppManagement/mobileApps/{newAppId}/updateRelationships
{
  "relationships": [{
    "@odata.type": "#microsoft.graph.mobileAppSupersedence",
    "targetId": "{oldAppId}",
    "supersedenceType": "replace"
  }]
}
```

This is the kind of thing that's easy to lose an hour to if you trust the documented navigation-property pattern at face value — the dedicated action endpoint isn't the first thing you'd reach for, and there's no error message steering you toward it; the call to the documented pattern just doesn't do anything.

## Targeting exactly one device

Intune app assignments target groups, not individual devices — by design, since managing per-device assignments doesn't scale. For a one-off "just this VM" situation, the tool for the job is an **Assignment Filter**: a rule evaluated against device properties (name, OS, model, etc.) that narrows down who in an assigned group actually receives the app.

```powershell
$filter = @{
    displayName = "Single-VM-Filter"
    platform    = "windows10AndLater"
    rule        = '(device.deviceName -eq "<vm-name>")'
} | ConvertTo-Json
Invoke-MgGraphRequest -Method POST `
  -Uri "https://graph.microsoft.com/beta/deviceManagement/assignmentFilters" `
  -Body $filter -ContentType "application/json"
```

Then assign the new app version to the existing device group as normal, but attach the filter with `include` mode so only devices matching the rule actually get it:

```powershell
$assignment = @{
    mobileAppAssignments = @(@{
        "@odata.type" = "#microsoft.graph.mobileAppAssignment"
        intent = "required"
        target = @{
            "@odata.type" = "#microsoft.graph.groupAssignmentTarget"
            groupId = "<groupId>"
            deviceAndAppManagementAssignmentFilterId   = "<filterId>"
            deviceAndAppManagementAssignmentFilterType = "include"
        }
    })
} | ConvertTo-Json -Depth 5
Invoke-MgGraphRequest -Method POST `
  -Uri "https://graph.microsoft.com/v1.0/deviceAppManagement/mobileApps/{newAppId}/assign" `
  -Body $assignment -ContentType "application/json"
```

The rest of the group is untouched; only the device matching the filter gets the new version on next sync. Once the fix was validated, removing the filter (or widening its rule) rolls the same supersedence chain out to the whole fleet on the team's own schedule.

## Takeaways

- **A vendor's server-side minimum-version enforcement can turn a "nice to update eventually" app into a hard outage with zero warning.** Worth knowing this before it happens, not during.
- **Intune's documented supersedence relationship pattern (cast + `/relationships`) doesn't reliably persist changes** — use the `/updateRelationships` action directly on the app.
- **Assignment Filters are the right tool for "just this one device," not manual per-device assignment.** They let you scope a change tightly now and widen it deliberately later, without ever re-touching the base group assignment.
