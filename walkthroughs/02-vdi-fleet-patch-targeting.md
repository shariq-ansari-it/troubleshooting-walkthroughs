# Patching a VDI Fleet Without Touching Everyone Else's Session

## TL;DR

A softphone/VDI client app started throwing "Update required — your desktop app is no longer supported" on one virtual desktop, while identical VMs in the same Intune group kept working fine. The app vendor enforces a minimum client version server-side; once a version falls below the cutoff, it stops working entirely rather than degrading gracefully. The fix needed a Win32 app supersedence chain in Intune plus a way to push the update to exactly one device without triggering an update — and a possible outage — across the whole fleet at once. Intune Assignment Filters solved the targeting problem; a Graph API quirk almost derailed the supersedence setup.

## Purpose of this Document

A case study and reference runbook for the problem summarised in the TL;DR above.

It is intentionally written to:

- Give the working Graph calls for supersedence and single-device targeting
- Keep the working path separate from the documented-but-broken supersedence pattern
- Help recognise the minimum-version outage pattern before it happens again

## Environment

- Azure Virtual Desktop VMs, all in one Intune device group
- Microsoft Intune — Win32 line-of-business apps, supersedence, Assignment Filters
- Microsoft Graph API (`beta` and `v1.0`), called via `Invoke-MgGraphRequest`
- VDI-aware softphone client shipped as two Win32 apps:
  - a main app that runs on the VM
  - a local plugin on the user's physical endpoint that offloads audio/video processing (without it calls still work, but quality suffers and the VM absorbs load it shouldn't)
- Vendor enforces a hard minimum supported client version server-side

---

## Correct End-to-End Process (Authoritative)

The goal: patch the one broken VM immediately, then plan the fleet-wide rollout separately and deliberately. Pushing to the whole group same-day risks breaking working sessions for everyone else, with no ability to stage or roll back gradually.

### 1. Set Up Supersedence on the New App Version

In Intune, "this new app version replaces that old one" is **supersedence**, configured on Win32 LOB apps. Call the dedicated action endpoint directly, with no cast (the documented navigation-property pattern doesn't persist — see [Issue 1](#issue-1--documented-supersedence-pattern-silently-doesnt-persist)):

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

### 2. Create an Assignment Filter for the One Device

Intune app assignments target groups, not individual devices — by design, since per-device assignments don't scale. An **Assignment Filter** is a rule evaluated against device properties (name, OS, model, etc.) that narrows down who in an assigned group actually receives the app.

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

### 3. Assign the New Version to the Existing Group With the Filter in `include` Mode

Assign to the existing device group as normal, attaching the filter so only matching devices get it:

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

Expected: the rest of the group is untouched; only the device matching the filter gets the new version on next sync.

### 4. Widen to the Fleet When Ready

Once the fix is validated, remove the filter (or widen its rule) to roll the same supersedence chain out to the whole fleet on your own schedule — without re-touching the base group assignment.

**If this all happens → the broken VM is patched now, and the fleet rollout is a deliberate later step.**

---

## Issues Encountered in This Case

### Issue 1 — Documented supersedence pattern silently doesn't persist

**Observed:** setting supersedence via the `relationships` navigation property with an OData type cast on the app object, as the Graph API documentation describes, did nothing — the relationship wasn't saved, and no error was returned.

**Root cause:** that path silently fails to persist the supersedence relationship.

**Fix:** call `/updateRelationships` directly on the app, no cast ([step 1](#1-set-up-supersedence-on-the-new-app-version)). Easy to lose an hour to — nothing steers you toward the action endpoint.

### Issue 2 — App refuses to run below the vendor's minimum version

**Observed:** "Update required — your desktop app is no longer supported" on one VM; identical VMs in the same Intune group still fine.

**Root cause:** the vendor enforces a minimum client version server-side. Below the cutoff the app doesn't nag — it stops working entirely.

**Fix:** push the newer version via supersedence, targeted to the affected device first ([steps 1–3](#1-set-up-supersedence-on-the-new-app-version)).

## Lessons Learned

- **A vendor's server-side minimum-version enforcement can turn a "nice to update eventually" app into a hard outage with zero warning.** Worth knowing this before it happens, not during.
- **Intune's documented supersedence relationship pattern (cast + `/relationships`) doesn't reliably persist changes** — use the `/updateRelationships` action directly on the app.
- **Assignment Filters are the right tool for "just this one device," not manual per-device assignment.** They let you scope a change tightly now and widen it deliberately later, without ever re-touching the base group assignment.

## Quick Reference Checklist

| If you see… | Do this |
|---|---|
| "Update required — your desktop app is no longer supported" | Vendor minimum-version cutoff — push the new version via supersedence |
| Supersedence via `relationships` + cast doesn't stick | `POST .../mobileApps/{newAppId}/updateRelationships`, no cast |
| Need to update one device without touching its group | Assignment Filter on `device.deviceName`, assignment with `include` mode |
| Fix validated on one device | Remove or widen the filter to roll out to the fleet |
