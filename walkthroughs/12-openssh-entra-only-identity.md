# Windows OpenSSH Can't Authenticate a Cloud-Only Entra Identity — and Other SSH Surprises

## Purpose of this Document

A case study and setup reference for getting direct SSH working on an Entra-joined Windows PC, as a lighter-weight alternative to a Tailscale-brokered RDP session and for scripted/automated tasks. The headline finding: Windows' OpenSSH server cannot authenticate a cloud-only Entra ID identity as a Windows security principal, in any username format.

It is intentionally written to:

- Give the setup sequence that worked
- Keep that path separate from the surprises hit along the way (a silent slow install, a firewall profile mismatch, the Entra identity limitation, background processes dying with the session)
- Record each with its symptom, cause and fix, so it can be recognised quickly next time
- Keep the measured Tailscale vs. direct LAN benchmark result as a concrete number

## Environment

- Windows PC, Microsoft Entra ID join (Azure AD-only, cloud-only identities)
- Windows OpenSSH Server (Win32-OpenSSH)
- Windows Firewall (Domain/Private/Public profiles)
- Tailscale mesh network
- WMI/CIM (`Win32_Process`)
- `iperf3` for throughput benchmarking

---

## Correct End-to-End Process (Authoritative)

### 1. Install OpenSSH Server and Let It Finish

Install the OpenSSH Server component with `Add-WindowsCapability`. It can run for a long time with no visible progress. Don't kill and restart it — check whether `TiWorker.exe` is still consuming CPU time over successive samples. See [Issue 1](#issue-1--the-install-looked-hung).

### 2. Check the Firewall Rule's Profile Against the Adapter's Profile

OpenSSH creates its own Windows Firewall rule, scoped to the **Private** profile. Check which profile the LAN-facing adapter is actually bound to — a virtual adapter may be classified as **Public**. See [Issue 2](#issue-2--reachable-over-tailscale-not-over-lan).

### 3. Authenticate as a Local Account, Not the Entra Identity

Set up public-key authentication (`administrators_authorized_keys`) for the **local** administrator account. A cloud-only Entra ID identity cannot authenticate to Win32-OpenSSH in any username format. See [Issue 3](#issue-3--key-based-auth-failing-with-invalid-user-no-matter-the-username-format).

### 4. Launch Long-Running Processes Fully Detached

Anything that must outlive the SSH session (e.g. an `iperf3` server) must be started outside the session's process tree. See [Issue 4](#issue-4--a-benchmark-process-kept-dying-when-the-ssh-session-closed).

```powershell
Invoke-CimMethod -ClassName Win32_Process -MethodName Create -Arguments @{
    CommandLine = "iperf3 -s"
}
```

### 5. Benchmark

Once measurement was reliable: Tailscale ran roughly **20% slower than a raw direct LAN connection**, even confirmed to be operating in genuine peer-to-peer mode rather than relayed through a DERP server. A reasonable, expected cost for the convenience of not managing direct network reachability — worth knowing as a concrete number rather than an assumption.

**If this all happens → direct SSH working on the Entra-joined PC via a local account.**

---

## Issues Encountered in This Case

### Issue 1 — The install looked hung

**Observed:** `Add-WindowsCapability` for the OpenSSH Server component ran for a long time with no visible progress, to the point of looking stalled.

**Root cause:** not hung — Windows Update/servicing work often runs invisibly through `TiWorker.exe`. Watching `TiWorker.exe`'s CPU-time deltas over successive samples confirmed it was still actively consuming CPU time: genuinely working, just slow and silent.

**Fix:** let it finish. Restarting would likely have meant starting the slow part over.

### Issue 2 — Reachable over Tailscale, not over LAN

**Observed:** the SSH server was reachable over the Tailscale mesh network but not from a plain LAN connection.

**Root cause:** the built-in Windows Firewall rule OpenSSH creates for itself was scoped to the **Private** network profile, while the LAN-facing network adapter (the vEthernet interface Hyper-V/WSL-style networking creates) was classified as **Public** by Windows.

**What didn't work:** "the firewall rule exists and looks correct" — the natural first check — doesn't reveal this without also checking which profile the adapter is bound to.

**Fix:** the rule itself isn't wrong — the mismatch is between the rule's profile (Private) and the adapter's classification (Public). Check which profile the LAN-facing adapter is actually bound to and resolve that mismatch.

### Issue 3 — Key-based auth failing with "invalid user", no matter the username format

**Observed:** public-key authentication against `administrators_authorized_keys` failed repeatedly with an "invalid user" style error.

**What didn't work:** every reasonable username format — bare username, UPN (`user@domain`), and `domain\user`.

**Root cause:** Windows' OpenSSH server (Win32-OpenSSH) fundamentally cannot resolve a **cloud-only Entra ID identity** (a user with no on-prem/hybrid AD presence at all) to a valid Windows security principal for SSH authentication. This isn't a configuration mistake — it's a real, documented gap in how Win32-OpenSSH's user resolution works relative to Entra-only identities, regardless of the username format presented.

**Fix:** authenticate as the **local** administrator account instead of the Entra identity — local accounts resolve normally.

### Issue 4 — A benchmark process kept dying when the SSH session closed

**Observed:** a background `iperf3` process started over the SSH connection (to benchmark Tailscale vs. direct LAN throughput) terminated the instant the parent SSH session disconnected.

**Root cause:** Windows ties child console processes to their parent session by default, so a normally backgrounded process doesn't survive its SSH session ending the way it would on Linux.

**Fix:** launch it as a **fully detached** process via WMI/CIM:

```powershell
Invoke-CimMethod -ClassName Win32_Process -MethodName Create -Arguments @{
    CommandLine = "iperf3 -s"
}
```

`Win32_Process.Create` starts a process that isn't a child of the calling session at all, so it survives the SSH connection closing.

## Lessons Learned

- **A cloud-only Entra ID account cannot authenticate to Windows OpenSSH Server as itself, in any username format — use a local account for SSH on Entra-only-joined machines**, and don't spend time trying different username formats expecting one to work
- **A slow-looking install isn't necessarily a hung one** — checking whether the relevant background process (`TiWorker.exe` for servicing operations) is still consuming CPU time is a fast way to tell "working slowly" from "actually stuck" before restarting something that would have finished on its own
- **Windows Firewall rules are scoped per network profile (Domain/Private/Public), and virtual adapters don't always get classified the way you'd expect** — check which profile the relevant adapter is bound to, not just whether the rule itself looks correct
- **A backgrounded process over SSH on Windows is still tied to its parent session by default** — use `Win32_Process.Create` (or an equivalent fully detached launch method) for anything that needs to outlive the SSH connection that started it
- **Tailscale (peer-to-peer, not DERP-relayed) measured ~20% slower than direct LAN** — an expected cost for not managing direct reachability

## Quick Reference Checklist

| If you see… | Do this |
|---|---|
| `Add-WindowsCapability` for OpenSSH Server appears stalled | Sample `TiWorker.exe` CPU time; if it's rising, let it finish |
| SSH reachable over Tailscale but not LAN | Firewall rule is Private-only; check the LAN adapter's profile (vEthernet may be Public) |
| "invalid user" with an Entra account in any format (bare, UPN, `domain\user`) | Cloud-only Entra identities can't authenticate to Win32-OpenSSH — use the local administrator account |
| Background process dies when the SSH session closes | Start it via `Invoke-CimMethod -ClassName Win32_Process -MethodName Create` |
