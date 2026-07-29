# Why `az login` Can't Query Intune (And What Does)

**Stack:** Azure CLI, Microsoft Graph API, Microsoft Graph PowerShell SDK, Intune/device management

## TL;DR

Trying to query Intune/device-management data (compliance state, managed devices, etc.) via Microsoft Graph, authenticated through the Azure CLI, fails with `AADSTS65002` — no matter what scope you request or what role the signed-in account holds. This isn't a permissions problem and consent won't fix it: the Azure CLI's own first-party application simply isn't pre-authorized by Microsoft for Intune/device-management Graph scopes. The Microsoft Graph PowerShell SDK's app registration *is* pre-authorized for exactly these scopes — switching to it is the actual fix, not a workaround for something wrong on your end.

## The symptom

```
az login --scope https://graph.microsoft.com/DeviceManagementManagedDevices.Read.All
```
or the equivalent token-acquisition call for that scope, fails with `AADSTS65002`, regardless of the signed-in user's role assignments, tenant admin consent settings, or the specific device-management scope requested. It looks like a permissions or consent problem. It reads like your account might be missing a role. It isn't either of those.

## Root cause

Every application that requests a Microsoft Graph scope has to be **pre-authorized by Microsoft** for the resource that scope belongs to — this is separate from tenant-level admin consent, which only controls whether *your organization* allows the app to use scopes it's already eligible to request. The Azure CLI's own backing application (a first-party Microsoft app, same one behind `az login` generally) is simply not on the pre-authorized list for Intune/device-management scopes. No amount of admin consent, Global Administrator role, or Conditional Access exemption changes that — it's a property of the CLI's app registration, not of your tenant or your account.

This is easy to lose time to, because `AADSTS65002` on its own doesn't say "this application isn't eligible for this scope" — it just presents as an auth failure, and the instinct is to go check your own account's permissions.

## The fix: a different app, not a different flag

The **Microsoft Graph PowerShell SDK** ships with its own first-party app registration ("Microsoft Graph Command Line Tools"), and that one *is* pre-authorized for device-management scopes:

```powershell
Connect-MgGraph -Scopes 'DeviceManagementManagedDevices.Read.All'
```

Same tenant, same signed-in user, same requested scope — the only variable that changes is which application is making the request, and that's the variable that actually matters here.

## A related gotcha once you're using Graph PowerShell

Some device-management detail — compliance-policy setting-level failures, for example — isn't exposed through a nicely named stable `Get-Mg...` cmdlet at v1.0. Rather than guessing at beta cmdlet names that may or may not exist, it's more reliable to hit the beta REST endpoint directly through the SDK's generic request cmdlet:

```powershell
Invoke-MgGraphRequest -Method GET `
  -Uri "https://graph.microsoft.com/beta/deviceManagement/managedDevices/{deviceId}/deviceCompliancePolicyStates/{policyStateId}/settingStates"
```

And a second, easy-to-miss gotcha: **each PowerShell invocation is its own fresh process.** If you run `Connect-MgGraph` in one `pwsh -Command` call and then try to query in a separate one, the session doesn't carry over — you'll get an unauthenticated error that looks unrelated to the actual cause. Any script that needs to query Graph has to call `Connect-MgGraph` and do the query in the *same* process/invocation, not a prior one. This applies whether you're running commands from an automated tool or your own interactive terminal, one command at a time — either way, each invocation starts clean.

## Takeaways

- **`AADSTS65002` from the Azure CLI against a Graph scope it can't use isn't a consent or role problem — it's an application pre-authorization problem, and no amount of tenant-side permission changes will fix it.**
- **When a specific first-party tool can't reach a specific Graph resource, check whether a *different* first-party Microsoft tool (Graph PowerShell SDK, Graph Explorer, etc.) is pre-authorized for it instead of assuming the resource itself is unreachable.**
- **For Graph functionality not yet exposed by a stable-named cmdlet, go straight to the beta REST endpoint via the SDK's generic request cmdlet rather than guessing at cmdlet names that might not exist.**
- **Interactive/browser-based auth flows and their resulting sessions don't survive across separate script invocations** — authenticate and query within the same process, every time.
