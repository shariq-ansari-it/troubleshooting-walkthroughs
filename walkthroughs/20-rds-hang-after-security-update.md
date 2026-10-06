# RDP and Console Logons Hang After a Security Update — and the Fix Windows Update Never Offered

## Purpose of this Document

A case study and recovery runbook for a Hyper-V guest running a multi-user line-of-business app (Sage 200 desktop client over RDP, with SQL Server) that stopped accepting logons — over RDP and through the Hyper-V console — while Hyper-V showed it running, heartbeat OK, low CPU. Low disk space was the first guess, because SQL logs had been cleared two days earlier (log growth on this kind of host is covered in [walkthrough 15](15-rds-performance-collapse.md)), but it had nothing to do with it. The real cause was a known Microsoft bug in the **September 2026 security update (KB5122871 on Server 2025, KB5122882 on Server 2022)** that can leave Remote Desktop Services hung. Microsoft's out-of-band fix is **offered only through the Update Catalog and WSUS, not Windows Update** — the server installed the faulty update twelve days *after* the fix came out and never received the fix.

It is intentionally written to:

- Give the complete path: get in, recover, prove the cause, install the fix, stop it recurring
- Keep that path separate from the red herrings and failed recovery attempts
- Record each roadblock with its symptom, cause and fix, so the pattern is recognised quickly after the next Patch Tuesday

## Environment

- Windows Server 2025 / 2022 (Hyper-V guest; Server 2019 also affected by the same known issue)
- Remote Desktop Services — multi-user Sage 200 desktop client over RDP
- SQL Server on the same guest
- Hyper-V host — VMConnect console, PowerShell Direct
- Patching straight from Windows Update (no WSUS approval step)

---

## Correct End-to-End Process (Authoritative)

### 1. Recognise the Symptom

- RDP logons fail for every user; existing sessions get slow, then stop responding.
- The Hyper-V console (VMConnect) logon fails too, so this isn't only a network or RDP listener problem.
- Hyper-V reports the VM as **Running**, heartbeat **OK**, CPU low. Nothing in the host UI looks wrong.

### 2. Get In With PowerShell Direct

**PowerShell Direct** goes over the Hyper-V VMBus, not the network, so it still works when RDP and the console logon don't:

```powershell
$vm   = "<VM name as shown in Hyper-V>"
$cred = Get-Credential   # local or domain admin inside the guest
Invoke-Command -VMName $vm -Credential $cred { Get-Volume | ft DriveLetter, SizeRemaining, Size }
```

Here the guest responded straight away and C: had ~15 GB free — ruling out the disk-space theory within the first few minutes ([Issue 1](#issue-1--low-disk-space-was-the-first-guess)).

### 3. Run a Quick Health Sweep

```powershell
Invoke-Command -VMName $vm -Credential $cred {
  "--- Sessions ---";     quser 2>&1
  "--- RDP services ---"; Get-Service TermService, SessionEnv, UmRdpService | ft Name, Status
  "--- Free RAM (GB) ---"; [math]::Round((Get-CimInstance Win32_OperatingSystem).FreePhysicalMemory/1MB,2)
  "--- Top memory ---";   Get-Process | sort WS -desc | select -first 5 Name, @{n='WS_GB';e={[math]::Round($_.WS/1GB,2)}}
}
```

Results in this case:

- Memory was fine (about half free; SQL Server wasn't the problem).
- All three RDP services reported **Running**.
- **`quser` hung and never returned.**

That combination is the key signal: the session manager has stopped responding even though every RDS service still shows as Running — see [Issue 2](#issue-2--rds-services-show-running-but-quser-hangs).

### 4. Recover the VM: Turn Off, Then Start

Graceful options all failed ([Issue 3](#issue-3--graceful-and-forced-shutdowns-never-complete)). What worked:

```powershell
Stop-VM -TurnOff   # Hyper-V "Turn Off", then Start
```

Clean boot, and console and RDP logons were fine immediately.

### 5. Confirm SQL Server Recovered

Turn Off is the equivalent of pulling the power, so SQL Server runs crash recovery on the next boot. Check that every database comes back `ONLINE` before you let users in:

```sql
SELECT name, state_desc FROM sys.databases;
```

### 6. Build the Event Timeline

**Pick the time window from uptime.** The last boot before the incident was the Sunday night the monthly updates finished installing — that gave the start of the window; the forced power-off gave the end. Take the window from the last boot, not from when users first complained.

```powershell
$start = Get-Date "<last boot before incident>"
$end   = Get-Date "<just before the forced power-off>"
$logs  = 'System','Application',
         'Microsoft-Windows-TerminalServices-LocalSessionManager/Operational',
         'Microsoft-Windows-TerminalServices-RemoteConnectionManager/Operational',
         'Microsoft-Windows-RemoteDesktopServices-RdpCoreTS/Operational',
         'Microsoft-Windows-User Profile Service/Operational'
$ev = foreach ($l in $logs) {
  Get-WinEvent -FilterHashtable @{LogName=$l; StartTime=$start; EndTime=$end} -EA 0 |
    Where-Object { $_.Level -in 1,2,3 -or $l -like '*TerminalServices*' }
}
$ev | Sort-Object TimeCreated |
  Select-Object TimeCreated, LogName, ProviderName, Id, LevelDisplayName, Message |
  Export-Csv C:\Temp\rds-hang.csv -NoTypeInformation
```

The timeline, once sorted:

| Time | Event |
|---|---|
| Morning | Normal logons (LSM 21/22). The last successful logon was about 45 minutes before the first failure. |
| T+0 | **Winlogon 6005:** *Remote Desktop Configuration (SessionEnv) is taking a long time to handle the Reconnect notification.* This was the first failure. |
| T+17 to T+50 min | SCM 7011 / 7046: IP Helper, Network List Service, Network Connection Broker and RDP UserMode Port Redirector repeatedly stopped responding. |
| T+20 min | Application Hang 1002 in the LOB desktop client. |
| T+40 min onward | New logons start (LSM 41) but never complete (no 21/22). |
| Later | Restart initiated (1074) but never finished; Kernel-Power 41 / 6008 at the forced power-off. |

What *wasn't* there mattered just as much: no Event 2013 (low disk) and no SQL 9002/1105 (log full). The disk-space theory was dead.

### 7. Rule Out Third-Party Software

A filter driver or network shim can cause exactly this kind of cascading service hang, so check before blaming Microsoft:

```powershell
fltmc                                                                      # only in-box filters present
Get-NetAdapterBinding | ? Enabled | ft Name, DisplayName, ComponentID      # only standard Microsoft bindings
```

Both were clean.

### 8. Check Installed Updates and Build

This is the check that gave the answer:

```powershell
Get-HotFix | Sort InstalledOn -Desc | select -First 10
Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion' | select ProductName, CurrentBuild, UBR
Get-HotFix -Id KB5129235, KB5129237 -EA 0
```

The September security update had been installed and activated by the Sunday reboot. The build was **26100.33438**, lower than the fixed build. The out-of-band fix was **not installed** — see [Issue 4](#issue-4--known-bug-in-the-september-2026-security-update) and [Issue 5](#issue-5--the-fix-is-not-offered-through-windows-update).

### 9. Install the Out-of-Band Fix

Install the matching out-of-band update out of hours (restart required), manually from the **Microsoft Update Catalog** or approved in **WSUS**. The fixes are cumulative, so they install directly on top of the faulty update with no need to remove it first.

| OS | Faulty update | Out-of-band fix (14 Sep 2026) | Fixed build |
|---|---|---|---|
| Windows Server 2025 | KB5122871 | **KB5129235** | 26100.33451 |
| Windows Server 2022 | KB5122882 | **KB5129237** | 20348.5631 |
| Windows Server 2019 | KB5122876 | KB5129238 | — |

Until the fix is installed the hang can come back. The recovery is the same: PowerShell Direct to confirm it's this problem (`quser` hangs), then Turn Off → Start.

### 10. Confirm the Build

- Server 2025: `Get-HotFix -Id KB5129235` and `UBR` ≥ 33451
- Server 2022: `KB5129237` / `UBR` ≥ 5631

**If this all happens → RDS is on the fixed build and the hang shouldn't return.**

### 11. Prevent a Repeat

- **Check Release Health for known issues before (or right after) every monthly patch cycle,** especially for RDS hosts. Out-of-band fixes for regressions often don't come through the channel that delivered the regression.
- **If you patch from Windows Update alone, OOB fixes need a manual step.** Catalog download or WSUS approval.
- **Patch one RDS host first and let it run a working day** before rolling out to the rest. Here, one VM on the host was on the new OS and the others hadn't been patched yet. Treat that as an accidental canary and keep the others off the faulty update.

---

## Issues Encountered in This Case

### Issue 1 — Low disk space was the first guess

**Observed:** logons failing; SQL logs had been cleared two days earlier, so low disk space was everyone's first suspect.

**Root cause:** unrelated. C: had ~15 GB free, and the event logs had no Event 2013 (low disk) and no SQL 9002/1105 (log full).

**Fix:** none needed for the hang. SQL transaction log growth is what caused the disk-space scare in the first place — diagnosis and the safe fix are in [walkthrough 15](15-rds-performance-collapse.md).

### Issue 2 — RDS services show Running but `quser` hangs

**Observed:** `TermService`, `SessionEnv` and `UmRdpService` all **Running**; `quser` never returned.

**Root cause:** the session manager has hung. Service status told you nothing useful here.

**Fix:** treat a hanging `quser` as the diagnostic signal and go to recovery ([step 4](#4-recover-the-vm-turn-off-then-start)).

### Issue 3 — Graceful and forced shutdowns never complete

**Observed:** escalating attempts to restart the VM:

| Attempt | Result |
|---|---|
| Hyper-V **Shut Down** | Refused: *"machine is locked and cannot be shut down without the force option"* (`0x800704F7`). A locked user session blocks a graceful shutdown. |
| `Restart-Computer -Force` via PowerShell Direct | Event 1074 was logged (restart initiated), but the shutdown never finished and uptime kept climbing. |
| `Stop-VM -Force` | Timed out (`0x800705B4`). |
| `Stop-VM -TurnOff` (Hyper-V **Turn Off**), then Start | **Worked.** Clean boot, and console and RDP logons were fine immediately. |

**Root cause:** Hyper-V Shut Down fails with `0x800704F7` when a locked user session exists, and in a hung-RDS state even a forced graceful shutdown may never finish.

**Fix:** Turn Off is the last resort — then check SQL databases ([step 5](#5-confirm-sql-server-recovered)).

### Issue 4 — Known bug in the September 2026 security update

**Observed:** first failure was Winlogon 6005 (SessionEnv slow to handle Reconnect), followed by cascading SCM 7011/7046 service hangs; build 26100.33438 with the September update installed and no out-of-band fix.

**Root cause:** a known issue in the September 2026 security updates. Microsoft's wording is that RDS "might become unstable, causing RDP connection and sign-in failures or servers to become unresponsive during Remote Desktop configuration". MMC, the RDS Licensing Diagnoser and File Explorer can also stop responding.

**Fix:** install the out-of-band update ([step 9](#9-install-the-out-of-band-fix)).

### Issue 5 — The fix is not offered through Windows Update

**Observed:** the faulty update was installed **twelve days after** the fix was published, and the fix was never installed.

**Root cause:** the fix's KB page says the out-of-band update is **not offered through Windows Update**. It's only available from the **Microsoft Update Catalog and WSUS**, and in WSUS it still needs approving. Any server patched straight from Windows Update gets the September update with the bug, while the fix waits somewhere nobody is looking.

So this wasn't just bad luck, and it will happen again. Every other Server 2022/2025 machine patched the same way will get the faulty update and not the fix, unless someone installs the fix deliberately.

**Fix:** install from Catalog/WSUS ([step 9](#9-install-the-out-of-band-fix)) and add the manual step to the patch cycle ([step 11](#11-prevent-a-repeat)).

### Issue 6 — Forgotten seven-month-old Hyper-V checkpoint (found along the way)

**Observed:** *Edit Disk* greyed out and the VM disk couldn't be expanded. The differencing disk (`.avhdx`) was ~155 GB against a 24 GB parent, so almost everything since the checkpoint was in the child disk.

**Root cause:** a forgotten Hyper-V checkpoint, seven months old. Unrelated to the hang, still worth fixing. Merging it needs roughly the size of the child disk in free host space, and a merge that fills the host volume **pauses every VM on that volume**.

**Fix (plan):**

1. Free host space first (e.g. `Move-VMStorage` a powered-off VM to another volume)
2. Delete the checkpoint out of hours and wait for the merge
3. `Resize-VHD` and extend the guest volume — check for a recovery partition sitting after C:

Also: **don't take a "safety" checkpoint before patching** when host space is already tight.

### Issue 7 — Other gotchas during the investigation

- **Console QuickEdit:** if a PowerShell window title starts with "Select", output is paused. Press Esc before assuming the command has hung.
- **Files copied over the RDP clipboard can arrive as all zero bytes** despite showing the right size. Check the file hash, or zip it first.

## Lessons Learned

- **Rule out the obvious suspect fast and with evidence.** Two minutes of PowerShell Direct killed the disk-space theory that everyone had started with.
- **PowerShell Direct (`Invoke-Command -VMName`) is the way in** when RDP and the console both fail. It doesn't touch the guest's network stack.
- **If a service says "Running", that doesn't mean it's responding.** A query that hangs (`quser`) is a much stronger signal than any service status.
- **Take the investigation window from uptime** (last boot), not from when users first complained.
- **After Patch Tuesday, the fix for a regression may not arrive through the same channel as the regression.** If you patch from Windows Update alone, you get Microsoft's bugs automatically and have to fetch its fixes yourself.
- **Incidents surface old debt.** A seven-month-old checkpoint and unmanaged SQL log growth weren't the cause, but either could have been the next outage.

## Quick Reference Checklist

| If you see… | Do this |
|---|---|
| RDP and VMConnect logons both fail, VM Running / heartbeat OK | PowerShell Direct: `Invoke-Command -VMName` |
| RDS services Running but `quser` hangs | Session manager hung → Turn Off → Start |
| Hyper-V Shut Down fails with `0x800704F7` | Locked user session — escalate |
| `Stop-VM -Force` times out (`0x800705B4`) | `Stop-VM -TurnOff`, then Start |
| After Turn Off | `SELECT name, state_desc FROM sys.databases;` — all `ONLINE` before users return |
| Winlogon 6005 (SessionEnv / Reconnect) then SCM 7011/7046 | Check for September 2026 update without the OOB fix |
| Server 2025 `UBR` < 33451 / Server 2022 `UBR` < 5631 | Install KB5129235 / KB5129237 from Catalog or WSUS |
| *Edit Disk* greyed out | Look for an old checkpoint (`.avhdx`) — free host space before merging |
| PowerShell window title starts with "Select" | Press Esc — output is paused |
| File copied over RDP clipboard looks right but is empty | Check the hash, or zip it first |

## References

- [KB5129235 — Windows Server 2025 out-of-band update (Microsoft Support)](https://support.microsoft.com/en-us/servicing/os/windows-server/2026/09/kb5129235-windows-server-2025-update)
- [Windows Server 2022 known issues (Microsoft Release Health)](https://learn.microsoft.com/en-us/windows/release-health/status-windows-server-2022)
- [Microsoft releases emergency Windows updates to fix RDS failures (BleepingComputer)](https://www.bleepingcomputer.com/news/microsoft/microsoft-releases-emergency-windows-updates-to-fix-rds-failures/)
