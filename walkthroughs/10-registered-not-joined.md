# Stuck at "Registered," Not "Joined" — the Fix Was a Link, Not a Network Fix

**Stack:** Windows 11, Microsoft Entra ID join, WinInet, third-party antivirus network filtering

## TL;DR

A new laptop wouldn't complete Microsoft Entra join. The visible error decoded to a WinInet name-resolution failure, which sent the investigation down a network/security-software path — a third-party antivirus's network filter driver, proxy configuration, IPv6, NRPT, hosts file, Windows Firewall — all correctly ruled out one at a time. The actual cause: the "Access work or school → Connect" flow the user had gone through only adds an SSO account; it doesn't perform a real Entra Join at all. The genuine join trigger was a separate, easy-to-miss link — "Join this device to Microsoft Entra ID" — at the bottom of that same settings page.

## The symptom

Error code `-2145833241`, which decodes to `0x80192EE7` — a WinInet error corresponding to "name not resolved" (commonly surfaced as error 12007 in browser-facing contexts). On the surface, that's a DNS/network-reachability problem, and Entra join genuinely does depend on reaching several Microsoft endpoints, so treating it as a network issue first was a reasonable call.

## Ruling things out, one at a time

- **Third-party antivirus network filtering.** The laptop had McAfee LiveSafe installed, which is known to install its own network filter driver capable of interfering with traffic in non-obvious ways. Fully removed via MCPR (McAfee's dedicated removal tool, since a normal uninstall often leaves the filter driver behind). No change.
- **Proxy configuration.** Checked WinHTTP/WinINet proxy settings for anything redirecting or blackholing traffic. Clean.
- **IPv6 stack issues.** A real, if less common, cause of selective name-resolution failures on some networks. Ruled out.
- **NRPT (Name Resolution Policy Table).** Checked for any policy-based DNS override rules that might be misrouting specific name lookups. None present.
- **Hosts file.** Checked for a stray entry redirecting a Microsoft endpoint. Clean.
- **Windows Firewall.** Checked outbound rules for anything blocking the relevant traffic. Nothing blocking.
- **Network switch test.** Connected the laptop to a mobile hotspot instead of the office network entirely, removing every corporate network variable (firewall, proxy, DNS, NRPT, AV filtering on that path) in one move. The join **still failed identically.** That result alone should have shifted suspicion away from "network problem" much earlier than it did.

## The actual cause

The user had gone through **Settings → Accounts → Access work or school → Connect**, signed in, and the flow appeared to complete. What that flow actually does is add an **organizational SSO account** to Windows — useful for things like syncing a work identity into apps — but it is a different operation from actually joining the device to Entra ID. The device stayed in a partial, "Registered" state rather than reaching "Joined."

The real trigger for a full Entra Join is a separate link, easy to miss because it sits at the bottom of the same "Access work or school" page rather than being the obvious primary action: **"Join this device to Microsoft Entra ID."** Clicking through that link (a distinct flow from the "Connect" button used first) is what actually performs the join.

Because the SSO-account flow "succeeds" without error, there was no failure signal pointing at the real gap — `dsregcmd /status` (which would have shown `AzureAdJoined: NO` immediately) wasn't checked until quite late in the investigation, after most of the network-side theories had already been exhausted.

## Takeaways

- **Check `dsregcmd /status` at the start of any Entra-join-shaped problem, not near the end.** It directly answers "Joined vs. Registered vs. neither" in one command and would have shown the real state immediately.
- **A network-switch test (e.g., a mobile hotspot) that reproduces the exact same failure is strong evidence the problem isn't network-related at all.** Worth running early — it collapses an entire branch of investigation in one step, rather than ruling network causes out individually.
- **"Access work or school → Connect" and "Join this device to Microsoft Entra ID" are two different operations that live on the same settings page and are easy to conflate** — especially since the first one visibly "succeeds," giving no indication that a further step is needed.
- **An error code that decodes to a network-layer failure doesn't guarantee the root cause is actually in the network layer** — it can just as easily be an application-level flow that never reached the point where a real network call was attempted.
