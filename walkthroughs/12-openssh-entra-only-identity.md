# Windows OpenSSH Can't Authenticate a Cloud-Only Entra Identity — and Other SSH Surprises

**Stack:** Windows OpenSSH Server, Microsoft Entra ID (Azure AD-only join), Windows Firewall, Tailscale, WMI/CIM

## TL;DR

Setting up direct SSH access to an Entra-joined PC (as an alternative to routing everything through a mesh-VPN-brokered RDP session) turned up a genuinely surprising, documented limitation: Windows' OpenSSH server cannot authenticate a cloud-only Entra ID identity as a Windows security principal, in any username format. Along the way: a slow feature install that looked hung but wasn't, a firewall rule scoped to the wrong network profile, and a background benchmark process that kept dying the moment its parent SSH session closed.

## Goal

Get direct SSH working on an Entra-joined Windows PC, mainly to compare it against an existing Tailscale-brokered RDP setup and to have a lighter-weight remote-access option for scripted/automated tasks.

## Gotcha 1: the install looked hung

`Add-WindowsCapability` for the OpenSSH Server component ran for a long time with no visible progress, to the point of looking stalled. Rather than killing and restarting it, checked whether it was actually still doing something by watching `TiWorker.exe`'s CPU-time deltas over successive samples (Windows Update/servicing work often runs invisibly through TiWorker) — confirmed it was still actively consuming CPU time, i.e., genuinely working, just slow and silent. Let it finish rather than restarting it, which would likely have meant starting the slow part over.

## Gotcha 2: reachable over Tailscale, not over LAN

Once installed, the SSH server was reachable over the Tailscale mesh network but not from a plain LAN connection. Root cause: the built-in Windows Firewall rule OpenSSH creates for itself was scoped to the **Private** network profile, while the LAN-facing network adapter (the vEthernet interface Hyper-V/WSL-style networking creates) was classified as **Public** by Windows — a classification mismatch that's easy to miss since "the firewall rule exists and looks correct" is the natural first check, without also checking which profile the relevant adapter is actually bound to.

## Gotcha 3: key-based auth failing with "invalid user," no matter the username format

Public-key authentication against `administrators_authorized_keys` failed repeatedly with an "invalid user" style error, tried against every reasonable username format: bare username, UPN (`user@domain`), and `domain\user`. None worked.

**Root cause — and the most interesting finding of the whole exercise:** Windows' OpenSSH server (Win32-OpenSSH) fundamentally cannot resolve a **cloud-only Entra ID identity** (a user with no on-prem/hybrid AD presence at all) to a valid Windows security principal for SSH authentication purposes. This isn't a configuration mistake to fix — it's a real, documented gap in how Win32-OpenSSH's user-resolution works relative to Entra-only identities, regardless of which username format is presented. The workaround was to authenticate as the **local** administrator account instead of the Entra identity — which works fine, since local accounts resolve normally.

## Gotcha 4: a benchmark process kept dying when the SSH session closed

Running a background `iperf3` process over the new SSH connection (to benchmark Tailscale vs. direct LAN throughput) kept terminating the instant the parent SSH session disconnected — Windows ties child console processes to their parent session by default, so a normal backgrounded process doesn't survive its SSH session ending the way it would on Linux. Fixed by launching it as a **fully detached** process via WMI/CIM instead:

```powershell
Invoke-CimMethod -ClassName Win32_Process -MethodName Create -Arguments @{
    CommandLine = "iperf3 -s"
}
```

`Win32_Process.Create` starts a process that isn't a child of the calling session at all, so it survives the SSH connection closing.

## The actual benchmark result

Once measurement was reliable: Tailscale ran roughly **20% slower than a raw direct LAN connection**, even confirmed to be operating in genuine peer-to-peer mode rather than relayed through a DERP server. A reasonable, expected cost for the convenience of not managing direct network reachability — worth knowing as a concrete number rather than an assumption.

## Takeaways

- **A cloud-only Entra ID account cannot authenticate to Windows OpenSSH Server as itself, in any username format — use a local account for SSH on Entra-only-joined machines**, and don't spend time trying different username formats expecting one of them to work.
- **A slow-looking install isn't necessarily a hung one** — checking whether the relevant background process (`TiWorker.exe` for servicing operations) is still consuming CPU time is a fast way to tell "working slowly" from "actually stuck" before restarting something that would have finished on its own.
- **Windows Firewall rules are scoped per network profile (Domain/Private/Public), and virtual adapters don't always get classified the way you'd expect** — check which profile the relevant adapter is actually bound to, not just whether the firewall rule itself looks correctly configured.
- **A backgrounded process over SSH on Windows is still tied to its parent session by default** — use `Win32_Process.Create` (or an equivalent fully-detached launch method) for anything that needs to outlive the SSH connection that started it.
