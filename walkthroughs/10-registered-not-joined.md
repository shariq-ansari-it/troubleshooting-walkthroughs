# Stuck at "Registered," Not "Joined" — the Fix Was a Link, Not a Network Fix

## Purpose of this Document

A case study for a new laptop that wouldn't complete Microsoft Entra join. The error decoded to a WinInet name-resolution failure, so every network and security-software cause was ruled out one at a time — but the real cause was that "Access work or school → Connect" only adds an SSO account. The actual join is a separate link at the bottom of the same page.

It is intentionally written to:

- Give the short path that actually gets the device joined
- Keep that path separate from the network-side theories that were all correctly ruled out
- Record each dead end, so the pattern is recognised quickly next time
- Put `dsregcmd /status` and the hotspot test at the start of the next Entra-join investigation, not the end

## Environment

- Windows 11 (new laptop)
- Microsoft Entra ID join
- WinInet
- Third-party antivirus with network filtering (McAfee LiveSafe)

---

## Correct End-to-End Process (Authoritative)

### 1. Check the Join State First

```
dsregcmd /status
```

Directly answers "Joined vs Registered vs neither" in one command. Here it would have shown immediately:

```
AzureAdJoined: NO
```

(See [Issue 5](#issue-5--dsregcmd-status-checked-late).)

### 2. If the Error Looks Network-Related, Run a Network-Switch Test Early

Connect the laptop to a mobile hotspot instead of the office network. If the join fails identically, the problem isn't the network — skip the individual network checks (see [Issue 3](#issue-3--hotspot-test-reproduced-the-failure)).

### 3. Use the Real Join Link

On **Settings → Accounts → Access work or school**, don't use the **Connect** button — that only adds an organizational SSO account (see [Issue 4](#issue-4--access-work-or-school--connect-doesnt-join-the-device)).

Use the separate link at the bottom of the same page:

**"Join this device to Microsoft Entra ID"**

That distinct flow is what actually performs the join.

**If this all happens → the device reaches "Joined," not just "Registered."**

---

## Issues Encountered in This Case

### Issue 1 — Error code decodes to a network failure

**Observed:** error code `-2145833241`, which decodes to `0x80192EE7` — a WinInet error corresponding to "name not resolved" (commonly surfaced as error 12007 in browser-facing contexts).

**Root cause:** on the surface a DNS/network-reachability problem, and Entra join genuinely does depend on reaching several Microsoft endpoints — so treating it as a network issue first was a reasonable call. But the flow used never performed a real join at all; an error that decodes to a network-layer failure doesn't guarantee the root cause is in the network layer.

### Issue 2 — Network and security-software causes, all ruled out

**What didn't work** (each checked and cleared):

- **Third-party antivirus network filtering** — McAfee LiveSafe, which is known to install its own network filter driver capable of interfering with traffic in non-obvious ways. Fully removed via MCPR (McAfee's dedicated removal tool, since a normal uninstall often leaves the filter driver behind). No change.
- **Proxy configuration** — WinHTTP/WinINet proxy settings checked for anything redirecting or blackholing traffic. Clean.
- **IPv6 stack issues** — a real, if less common, cause of selective name-resolution failures on some networks. Ruled out.
- **NRPT (Name Resolution Policy Table)** — checked for policy-based DNS override rules misrouting specific name lookups. None present.
- **Hosts file** — checked for a stray entry redirecting a Microsoft endpoint. Clean.
- **Windows Firewall** — outbound rules checked for anything blocking the relevant traffic. Nothing blocking.

### Issue 3 — Hotspot test reproduced the failure

**Observed:** on a mobile hotspot instead of the office network — removing every corporate network variable (firewall, proxy, DNS, NRPT, AV filtering on that path) in one move — the join **still failed identically.**

**Root cause:** that result alone should have shifted suspicion away from "network problem" much earlier than it did.

**Fix:** run this test early ([step 2](#2-if-the-error-looks-network-related-run-a-network-switch-test-early)) — it collapses an entire branch of investigation in one step.

### Issue 4 — "Access work or school → Connect" doesn't join the device

**Observed:** the user had gone through **Settings → Accounts → Access work or school → Connect**, signed in, and the flow appeared to complete.

**Root cause:** that flow adds an **organizational SSO account** to Windows — useful for things like syncing a work identity into apps — but it is a different operation from joining the device to Entra ID. The device stayed in a partial, "Registered" state rather than reaching "Joined." Because the SSO-account flow "succeeds" without error, there was no failure signal pointing at the real gap.

**Fix:** the separate **"Join this device to Microsoft Entra ID"** link at the bottom of the same page ([step 3](#3-use-the-real-join-link)) — easy to miss because it isn't the obvious primary action.

### Issue 5 — `dsregcmd /status` checked late

**Observed:** `dsregcmd /status` wasn't checked until quite late in the investigation, after most of the network-side theories had already been exhausted.

**Fix:** run it first ([step 1](#1-check-the-join-state-first)) — `AzureAdJoined: NO` would have shown the real state immediately.

## Lessons Learned

- **Check `dsregcmd /status` at the start of any Entra-join-shaped problem, not near the end.** It directly answers "Joined vs. Registered vs. neither" in one command and would have shown the real state immediately.
- **A network-switch test (e.g., a mobile hotspot) that reproduces the exact same failure is strong evidence the problem isn't network-related at all.** Worth running early — it collapses an entire branch of investigation in one step, rather than ruling network causes out individually.
- **"Access work or school → Connect" and "Join this device to Microsoft Entra ID" are two different operations that live on the same settings page and are easy to conflate** — especially since the first one visibly "succeeds," giving no indication that a further step is needed.
- **An error code that decodes to a network-layer failure doesn't guarantee the root cause is actually in the network layer** — it can just as easily be an application-level flow that never reached the point where a real network call was attempted.

## Quick Reference Checklist

| If you see… | Do this |
|---|---|
| Any Entra-join-shaped problem | `dsregcmd /status` first |
| `AzureAdJoined: NO` after "Access work or school → Connect" | Use "Join this device to Microsoft Entra ID" at the bottom of the same page |
| Error `-2145833241` / `0x80192EE7` (name not resolved) | Don't assume network — run a hotspot test first |
| Same failure on a mobile hotspot | Not a network problem — stop checking proxy/DNS/firewall |
| Suspected AV network filter driver | Remove with the vendor's dedicated removal tool (e.g. MCPR), not a normal uninstall |
