# Three "Noncompliant" Devices, Three Unrelated Root Causes

**Stack:** Microsoft Intune compliance policies, Microsoft Defender, MDM sync/heartbeat

## TL;DR

Three devices showed up noncompliant in Intune around the same time, and the working theory was that they shared a common cause — all three happened to be Windows 10. That theory got disproven cleanly by finding a fourth Windows 10 device on the identical OS build that *was* compliant. Once the "one shared cause" assumption was dropped, each device turned out to have a completely different, unrelated problem — including one actively-used machine that had silently stopped checking in with Intune for roughly nine months without anyone noticing.

## The starting theory, and why it fell apart

Three noncompliant devices, all Windows 10, naturally suggests looking for something common to Windows 10 specifically — an OS-version-scoped compliance policy behaving unexpectedly, a Windows 10 feature update issue, something like that.

That theory was tested directly: found another Windows 10 device, same OS build as the noncompliant ones, and checked its compliance state. It was compliant. Same OS, same build, different compliance outcome — that single comparison ruled out "something about Windows 10 itself" in one step, rather than requiring three separate parallel investigations to each independently rule out an OS-version cause.

## Device 1: silently unsynced for about nine months

The most operationally significant finding. This device was in **active daily use** by its assigned user — not sitting in a drawer, not decommissioned, genuinely being worked on every day. And yet it hadn't completed an Intune sync in approximately nine months.

That's a materially different kind of risk than a normal compliance failure: a device that's actively in use but invisible to Intune isn't receiving updated configuration profiles, isn't being evaluated against current compliance policy changes, and isn't receiving new security policy at all — while looking, to a user just doing their job day to day, completely normal. Nothing about the user's actual experience would have surfaced this; it was only visible by specifically checking last-sync timestamps in Intune rather than trusting that "device is in use" implies "device is checking in."

## Device 2: genuinely stale/dead

A more mundane case — a device that appeared to simply no longer be in active use at all. Confirmed via last-checkin data and treated as an offboarding/cleanup item rather than an active troubleshooting target.

## Device 3: a real, live compliance failure

**Microsoft Defender Real-Time Protection** was actually disabled on this one — a genuine, currently-true compliance violation, unlike the other two. Correctly flagged by the compliance policy for exactly the reason compliance policies exist. Resolution here was the straightforward one: re-enable Real-Time Protection and confirm the compliance state updates on next check-in.

## Why the three-unrelated-cause framing matters

The natural instinct when several devices fail the same check around the same time is to look for one shared explanation. That instinct is usually right — but confirming it costs very little (one comparison device, in this case) relative to what it costs to *not* check it and instead spend effort investigating three devices in parallel as if they're the same problem. Here, they genuinely weren't: one was an operational blind spot (silent sync failure on an active device), one was a lifecycle/offboarding matter, and one was an actual, current security-relevant failure. Each needed a completely different response, and treating them as a single incident would have masked at least two of the three.

## Takeaways

- **Test a shared-cause theory against a device that *doesn't* exhibit the symptom before investing effort investigating each affected device individually.** One clean comparison (same OS/build, different outcome) can save significant time versus three parallel investigations.
- **"Noncompliant" in Intune can mean several very different things — a real, current policy violation; a device that's stale/decommissioned; or a device that's actively used but has simply stopped checking in.** Each needs a different response, and lumping them together as "compliance issues" obscures which is which.
- **A device silently failing to sync for months while still being actively used is arguably the highest-risk state of the three** — it looks completely normal to the end user and to anyone glancing at the device in person, and only shows up by specifically checking last-sync/last-checkin timestamps rather than assuming active use implies active management.
- **Periodically auditing last-checkin times across your Intune fleet (not just reacting to compliance-flag alerts) is worth doing proactively** — a device can drift into this state with zero visible symptoms for the user, and zero alert unless something specifically checks for silence rather than for an active policy failure.
