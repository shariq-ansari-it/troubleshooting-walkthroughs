# Why `az login` Can't Query Intune (And What Does)

## TL;DR

Trying to query Intune/device-management data (compliance state, managed devices, etc.) via Microsoft Graph, authenticated through the Azure CLI, fails with `AADSTS65002` — no matter what scope you request or what role the signed-in account holds. This isn't a permissions problem and consent won't fix it: the Azure CLI's own first-party application simply isn't pre-authorized by Microsoft for Intune/device-management Graph scopes. The Microsoft Graph PowerShell SDK's app registration *is* pre-authorized for exactly these scopes — switching to it is the actual fix, not a workaround for something wrong on your end.

## Purpose of this Document

A case study and reference runbook for the problem summarised in the TL;DR above.

It is intentionally written to:

- Give the working path for getting Intune data out of Graph
- Keep that path separate from the permission/consent dead ends it's easy to lose time to
- Record each gotcha with its symptom, cause and fix, so it can be recognised quickly next time
- Be the single reference other walkthroughs link to for Graph PowerShell session behaviour

## Environment

- Azure CLI (`az login`)
- Microsoft Graph API (v1.0 and beta)
- Microsoft Graph PowerShell SDK ("Microsoft Graph Command Line Tools" app registration)
- Microsoft Intune / device management

---

## Correct End-to-End Process (Authoritative)

### 1. Don't Use the Azure CLI for Intune Graph Scopes

If the Azure CLI returns `AADSTS65002` for a device-management scope, stop — it's an application pre-authorization limit, not something to fix in the tenant. See [Issue 1](#issue-1--aadsts65002-from-the-azure-cli-on-device-management-scopes).

### 2. Connect With the Graph PowerShell SDK

The SDK's own first-party app registration ("Microsoft Graph Command Line Tools") *is* pre-authorized for device-management scopes:

```powershell
Connect-MgGraph -Scopes 'DeviceManagementManagedDevices.Read.All'
```

Same tenant, same signed-in user, same requested scope — the only variable that changes is which application is making the request, and that's the variable that actually matters here.

### 3. Use the Beta REST Endpoint for Detail Without a Stable Cmdlet

For data not exposed by a stable-named `Get-Mg...` cmdlet at v1.0 (e.g. compliance-policy setting-level failures), call the beta endpoint through the SDK's generic request cmdlet — see [Issue 2](#issue-2--no-stable-cmdlet-for-setting-level-compliance-detail):

```powershell
Invoke-MgGraphRequest -Method GET `
  -Uri "https://graph.microsoft.com/beta/deviceManagement/managedDevices/{deviceId}/deviceCompliancePolicyStates/{policyStateId}/settingStates"
```

### 4. Connect and Query in the Same Process

Run `Connect-MgGraph` and the query in the *same* PowerShell process/invocation, every time — see [Issue 3](#issue-3--connect-mggraph-session-doesnt-carry-over-between-invocations).

**If this all happens → Intune/device-management data returned from Graph.**

---

## Issues Encountered in This Case

### Issue 1 — `AADSTS65002` from the Azure CLI on device-management scopes

**Observed:**

```
az login --scope https://graph.microsoft.com/DeviceManagementManagedDevices.Read.All
```

or the equivalent token-acquisition call for that scope fails with `AADSTS65002`, regardless of the signed-in user's role assignments, tenant admin consent settings, or the specific device-management scope requested. It looks like a permissions or consent problem, and reads like your account might be missing a role. It isn't either of those.

**Root cause:** every application that requests a Microsoft Graph scope has to be **pre-authorized by Microsoft** for the resource that scope belongs to. This is separate from tenant-level admin consent, which only controls whether *your organization* allows the app to use scopes it's already eligible to request. The Azure CLI's own backing application (a first-party Microsoft app, the same one behind `az login` generally) is simply not on the pre-authorized list for Intune/device-management scopes. It's a property of the CLI's app registration, not of your tenant or your account.

`AADSTS65002` on its own doesn't say "this application isn't eligible for this scope" — it just presents as an auth failure, and the instinct is to go check your own account's permissions.

**What didn't work:** no amount of admin consent, Global Administrator role, or Conditional Access exemption changes it.

**Fix:** a different app, not a different flag — `Connect-MgGraph` ([step 2](#2-connect-with-the-graph-powershell-sdk)).

### Issue 2 — No stable cmdlet for setting-level compliance detail

**Observed:** some device-management detail — compliance-policy setting-level failures, for example — isn't exposed through a nicely named stable `Get-Mg...` cmdlet at v1.0.

**What didn't work:** guessing at beta cmdlet names that may or may not exist.

**Fix:** hit the beta REST endpoint directly with `Invoke-MgGraphRequest` ([step 3](#3-use-the-beta-rest-endpoint-for-detail-without-a-stable-cmdlet)).

### Issue 3 — `Connect-MgGraph` session doesn't carry over between invocations

**Observed:** `Connect-MgGraph` in one `pwsh -Command` call, then a query in a separate one, returns an unauthenticated error that looks unrelated to the actual cause.

**Root cause:** **each PowerShell invocation is its own fresh process.** The session doesn't carry over. This applies whether you're running commands from an automated tool or your own interactive terminal, one command at a time — either way, each invocation starts clean.

**Fix:** any script that needs to query Graph has to call `Connect-MgGraph` and do the query in the *same* process/invocation, not a prior one.

## Lessons Learned

- **`AADSTS65002` from the Azure CLI against a Graph scope it can't use isn't a consent or role problem — it's an application pre-authorization problem, and no amount of tenant-side permission changes will fix it.**
- **When a specific first-party tool can't reach a specific Graph resource, check whether a *different* first-party Microsoft tool (Graph PowerShell SDK, Graph Explorer, etc.) is pre-authorized for it instead of assuming the resource itself is unreachable.**
- **For Graph functionality not yet exposed by a stable-named cmdlet, go straight to the beta REST endpoint via the SDK's generic request cmdlet rather than guessing at cmdlet names that might not exist.**
- **Interactive/browser-based auth flows and their resulting sessions don't survive across separate script invocations** — authenticate and query within the same process, every time.

## Quick Reference Checklist

| If you see… | Do this |
|---|---|
| `AADSTS65002` from `az login` / az token call for a device-management scope | Don't chase roles or consent — switch to `Connect-MgGraph -Scopes '<scope>'` |
| No stable `Get-Mg...` cmdlet for the data you need | `Invoke-MgGraphRequest -Method GET -Uri "https://graph.microsoft.com/beta/..."` |
| Unauthenticated error right after a successful `Connect-MgGraph` | Connect and query in the same process/invocation |
| A first-party tool can't reach a Graph resource | Try another first-party tool (Graph PowerShell SDK, Graph Explorer) |
