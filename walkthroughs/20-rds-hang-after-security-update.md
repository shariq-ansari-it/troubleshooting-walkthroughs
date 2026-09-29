# RDP and Console Logons Hang After a Security Update — and the Fix Windows Update Never Offered

**Stack:** Windows Server 2025 / 2022, Remote Desktop Services, Hyper-V, PowerShell Direct, SQL Server

## TL;DR

A Hyper-V guest running a multi-user line-of-business app (Sage 200 desktop client over RDP, with SQL Server) stopped accepting logons: new RDP sessions failed, and so did logons through the Hyper-V console. Hyper-V said the VM was running, heartbeat OK, low CPU. Low disk space was the first guess, because SQL logs had been cleared two days earlier, but it turned out to have nothing to do with it. The real cause was a known Microsoft bug in the **September 2026 security update (KB5122871 on Server 2025, KB5122882 on Server 2022)** that can leave Remote Desktop Services hung. Microsoft fixed it in an out-of-band update released days later, but that fix is **offered only through the Update Catalog and WSUS, not Windows Update**. The server had installed the faulty update twelve days *after* the fix came out and never received the fix.

## The symptom

- RDP logons failed for every user; existing sessions got slow, then stopped responding.
- The Hyper-V console (VMConnect) logon failed too, so this wasn't only a network or RDP listener problem.
- Hyper-V reported the VM as **Running**, heartbeat **OK**, CPU low. Nothing in the host UI looked wrong.

## Getting in when RDP and the console are both dead

**PowerShell Direct** goes over the Hyper-V VMBus, not the network, so it still works when RDP and the console logon don't:

```powershell
$vm   = "<VM name as shown in Hyper-V>"
$cred = Get-Credential   # local or domain admin inside the guest
Invoke-Command -VMName $vm -Credential $cred { Get-Volume | ft DriveLetter, SizeRemaining, Size }
```

The guest responded straight away, and C: had ~15 GB free. That ruled out the disk-space theory within the first few minutes.

Next, a quick health sweep:

```powershell
Invoke-Command -VMName $vm -Credential $cred {
  "--- Sessions ---";     quser 2>&1
  "--- RDP services ---"; Get-Service TermService, SessionEnv, UmRdpService | ft Name, Status
  "--- Free RAM (GB) ---"; [math]::Round((Get-CimInstance Win32_OperatingSystem).FreePhysicalMemory/1MB,2)
  "--- Top memory ---";   Get-Process | sort WS -desc | select -first 5 Name, @{n='WS_GB';e={[math]::Round($_.WS/1GB,2)}}
}
```

- Memory was fine (about half free; SQL Server wasn't the problem).
- All three RDP services reported **Running**.
- **`quser` hung and never returned.**

That combination is the key signal. The session manager has stopped responding even though every RDS service still shows as Running. Here, service status told you nothing useful.

## Getting it back (escalating)

| Attempt | Result |
|---|---|
| Hyper-V **Shut Down** | Refused: *"machine is locked and cannot be shut down without the force option"* (`0x800704F7`). A locked user session blocks a graceful shutdown. |
| `Restart-Computer -Force` via PowerShell Direct | Event 1074 was logged (restart initiated), but the shutdown never finished and uptime kept climbing. |
| `Stop-VM -Force` | Timed out (`0x800705B4`). |
| `Stop-VM -TurnOff` (Hyper-V **Turn Off**), then Start | **Worked.** Clean boot, and console and RDP logons were fine immediately. |

Turn Off is the equivalent of pulling the power, so SQL Server runs crash recovery on the next boot. Check that every database comes back `ONLINE` before you let users in:

```sql
SELECT name, state_desc FROM sys.databases;
```

## Finding the cause

**Pick the time window from uptime.** The last boot before the incident was the Sunday night the monthly updates finished installing. That gave the start of the search window, and the forced power-off gave the end.

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

The timeline was clear once it was sorted:

| Time | Event |
|---|---|
| Morning | Normal logons (LSM 21/22). The last successful logon was about 45 minutes before the first failure. |
| T+0 | **Winlogon 6005:** *Remote Desktop Configuration (SessionEnv) is taking a long time to handle the Reconnect notification.* This was the first failure. |
| T+17 to T+50 min | SCM 7011 / 7046: IP Helper, Network List Service, Network Connection Broker and RDP UserMode Port Redirector repeatedly stopped responding. |
| T+20 min | Application Hang 1002 in the LOB desktop client. |
| T+40 min onward | New logons start (LSM 41) but never complete (no 21/22). |
| Later | Restart initiated (1074) but never finished; Kernel-Power 41 / 6008 at the forced power-off. |

What *wasn't* there mattered just as much: no Event 2013 (low disk) and no SQL 9002/1105 (log full). The disk-space theory was dead.

**Ruling out third-party software.** A filter driver or network shim can cause exactly this kind of cascading service hang, so it was worth checking before blaming Microsoft:

```powershell
fltmc                                                                      # only in-box filters present
Get-NetAdapterBinding | ? Enabled | ft Name, DisplayName, ComponentID      # only standard Microsoft bindings
```

Both were clean.

**The update check that gave the answer:**

```powershell
Get-HotFix | Sort InstalledOn -Desc | select -First 10
Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion' | select ProductName, CurrentBuild, UBR
Get-HotFix -Id KB5129235, KB5129237 -EA 0
```

The September security update had been installed and activated by the Sunday reboot. The build was **26100.33438**, lower than the fixed build. The out-of-band fix was **not installed**.

## Root cause

This is a known issue in the September 2026 security updates. Microsoft's wording is that RDS "might become unstable, causing RDP connection and sign-in failures or servers to become unresponsive during Remote Desktop configuration". MMC, the RDS Licensing Diagnoser and File Explorer can also stop responding.

| OS | Faulty update | Out-of-band fix (14 Sep 2026) | Fixed build |
|---|---|---|---|
| Windows Server 2025 | KB5122871 | **KB5129235** | 26100.33451 |
| Windows Server 2022 | KB5122882 | **KB5129237** | 20348.5631 |
| Windows Server 2019 | KB5122876 | KB5129238 | — |

The fixes are cumulative, so they install directly on top of the faulty update with no need to remove it first.

## The part that actually matters: why the fix wasn't there

The faulty update was installed **twelve days after** the fix was published. The reason is on the fix's KB page: the out-of-band update is **not offered through Windows Update**. It's only available from the **Microsoft Update Catalog and WSUS**, and in WSUS it still needs approving. Any server patched straight from Windows Update gets the September update with the bug, while the fix waits somewhere nobody is looking.

So this wasn't just bad luck, and it will happen again. Every other Server 2022/2025 machine patched the same way will get the faulty update and not the fix, unless someone installs the fix deliberately.

## Permanent fix

1. Install the matching out-of-band update out of hours (restart required), manually from the Catalog or approved in WSUS.
2. Confirm the build afterwards: `Get-HotFix -Id KB5129235` and `UBR` ≥ 33451 (Server 2025), or `KB5129237` / `UBR` ≥ 5631 (Server 2022).
3. Until the fix is installed the hang can come back. The recovery is the same: PowerShell Direct to confirm it's this problem (`quser` hangs), then Turn Off → Start.

## Prevention

- **Check Release Health for known issues before (or right after) every monthly patch cycle,** especially for RDS hosts. Out-of-band fixes for regressions often don't come through the channel that delivered the regression.
- **If you patch from Windows Update alone, OOB fixes need a manual step.** Catalog download or WSUS approval.
- **Patch one RDS host first and let it run a working day** before rolling out to the rest. Here, one VM on the host was on the new OS and the others hadn't been patched yet. Treat that as an accidental canary and keep the others off the faulty update.

## Found along the way (unrelated to the hang, still worth fixing)

- **A forgotten Hyper-V checkpoint, seven months old.** The differencing disk (`.avhdx`) was ~155 GB against a 24 GB parent, so almost everything since the checkpoint was in the child disk. That's why *Edit Disk* was greyed out and the VM disk couldn't be expanded. Merging it needs roughly the size of the child disk in free host space, and a merge that fills the host volume **pauses every VM on that volume**. The plan: free host space first (e.g. `Move-VMStorage` a powered-off VM to another volume), delete the checkpoint out of hours, wait for the merge, then `Resize-VHD` and extend the guest volume. Check for a recovery partition sitting after C:. Also: **don't take a "safety" checkpoint before patching** when host space is already tight.
- **SQL transaction log growth.** This is what caused the disk-space scare in the first place. Fix the recovery model and log backups rather than deleting log files:

  ```sql
  SELECT name, recovery_model_desc, log_reuse_wait_desc FROM sys.databases;
  DBCC SQLPERF(LOGSPACE);
  ```

  **Never delete `.ldf` files to "clear SQL logs".** Shrink through SQL, or remove old log backups and ERRORLOGs only.

## Gotchas

- **PowerShell Direct (`Invoke-Command -VMName`) is the way in** when RDP and the console both fail. It doesn't touch the guest's network stack.
- **`quser` hanging while RDS services show "Running" means the session manager has hung.** Don't trust service status here.
- **Hyper-V Shut Down fails with `0x800704F7`** when a locked user session exists. And in a hung-RDS state, even forced graceful shutdown may never finish. Turn Off is the last resort.
- **Console QuickEdit:** if a PowerShell window title starts with "Select", output is paused. Press Esc before assuming the command has hung.
- **Files copied over the RDP clipboard can arrive as all zero bytes** despite showing the right size. Check the file hash, or zip it first.
- **Take the investigation window from uptime** (last boot), not from when users first complained.

## Takeaways

- **Rule out the obvious suspect fast and with evidence.** Two minutes of PowerShell Direct killed the disk-space theory that everyone had started with.
- **If a service says "Running", that doesn't mean it's responding.** A query that hangs (`quser`) is a much stronger signal than any service status.
- **After Patch Tuesday, the fix for a regression may not arrive through the same channel as the regression.** If you patch from Windows Update alone, you get Microsoft's bugs automatically and have to fetch its fixes yourself.
- **Incidents surface old debt.** A seven-month-old checkpoint and unmanaged SQL log growth weren't the cause, but either could have been the next outage.

## References

- [KB5129235 — Windows Server 2025 out-of-band update (Microsoft Support)](https://support.microsoft.com/en-us/servicing/os/windows-server/2026/09/kb5129235-windows-server-2025-update)
- [Windows Server 2022 known issues (Microsoft Release Health)](https://learn.microsoft.com/en-us/windows/release-health/status-windows-server-2022)
- [Microsoft releases emergency Windows updates to fix RDS failures (BleepingComputer)](https://www.bleepingcomputer.com/news/microsoft/microsoft-releases-emergency-windows-updates-to-fix-rds-failures/)
