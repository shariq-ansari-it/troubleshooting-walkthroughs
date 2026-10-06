# Three "Noncompliant" Devices, Three Unrelated Root Causes

## Purpose of this Document

A case study in triaging several Intune devices that went noncompliant around the same time. The working theory — one shared cause, since all three were Windows 10 — was disproven by a single compliant comparison device on the identical build, and each device turned out to have a different, unrelated problem, including an actively used machine that had silently stopped checking in with Intune for roughly nine months.

It is intentionally written to:

- Give the triage sequence that separates a shared cause from unrelated ones quickly
- Keep that sequence separate from the wrong theory that was tested and dropped
- Record each device's actual cause and the response it needed
- Help recognise the three different meanings of "noncompliant" next time

## Environment

- Microsoft Intune compliance policies
- Windows 10 devices (same OS build)
- Microsoft Defender (Real-Time Protection)
- MDM sync / heartbeat (last-sync and last-checkin timestamps)

---

## Correct End-to-End Process (Authoritative)

### 1. Test the Shared-Cause Theory Against a Compliant Device

Several devices failing the same check at the same time naturally suggests one shared explanation — here, something about Windows 10 (an OS-version-scoped compliance policy behaving unexpectedly, a Windows 10 feature update issue).

Before investigating each device, find a device that has the same suspected common factor (same OS, same build) and check its compliance state. If it's compliant, the shared factor isn't the cause. One comparison rules it out in a single step, rather than three parallel investigations each independently ruling out an OS-version cause. See [Issue 1](#issue-1--the-one-shared-cause-theory).

### 2. Check Last-Sync / Last-Checkin Timestamps for Each Device

Don't trust that "device is in use" implies "device is checking in". Check each device's last-sync timestamp in Intune. This separates:

- an active device that has stopped syncing — see [Issue 2](#issue-2--device-1-silently-unsynced-for-about-nine-months)
- a device no longer in use at all — see [Issue 3](#issue-3--device-2-genuinely-staledead)

### 3. Check for a Real, Current Policy Violation

For devices that are checking in, look at what the compliance policy is actually flagging — e.g. Microsoft Defender Real-Time Protection disabled. See [Issue 4](#issue-4--device-3-a-real-live-compliance-failure).

### 4. Respond per Device, Not per Incident

| Category | Response |
|---|---|
| Actively used, silently not syncing | Operational blind spot — not a policy failure; it isn't being managed at all |
| Stale / no longer in use | Offboarding / cleanup item |
| Real, current violation | Fix the violation, confirm compliance updates on next check-in |

Treating the three as a single incident would have masked at least two of the three.

**If this all happens → each device gets the response its actual cause needs.**

---

## Issues Encountered in This Case

### Issue 1 — The one-shared-cause theory

**Observed:** three noncompliant devices, all Windows 10, around the same time.

**What didn't work:** assuming something common to Windows 10 — an OS-version-scoped compliance policy, a Windows 10 feature update issue.

**Root cause:** there was no shared cause. Another Windows 10 device on the same OS build as the noncompliant ones was compliant — same OS, same build, different compliance outcome — which ruled out "something about Windows 10 itself".

**Fix:** drop the shared-cause assumption and triage each device individually.

### Issue 2 — Device 1: silently unsynced for about nine months

**Observed:** the device was in **active daily use** by its assigned user — not in a drawer, not decommissioned — yet it hadn't completed an Intune sync in approximately nine months.

**Root cause:** a silent sync failure on an active device. Nothing about the user's experience would have surfaced it; it was only visible by specifically checking last-sync timestamps in Intune.

**Why it matters:** a materially different risk from a normal compliance failure. A device that's in use but invisible to Intune isn't receiving updated configuration profiles, isn't being evaluated against current compliance policy changes, and isn't receiving new security policy at all — while looking completely normal to the user.

**Fix:** treat as an operational blind spot rather than a policy failure — the problem is that the device isn't being managed at all, not what a policy says about it.

### Issue 3 — Device 2: genuinely stale/dead

**Observed:** a device that appeared to simply no longer be in active use.

**Root cause:** confirmed via last-checkin data — the device was no longer in use.

**Fix:** treated as an offboarding/cleanup item rather than an active troubleshooting target.

### Issue 4 — Device 3: a real, live compliance failure

**Observed:** **Microsoft Defender Real-Time Protection** was disabled.

**Root cause:** a genuine, currently true compliance violation — correctly flagged by the compliance policy for exactly the reason compliance policies exist.

**Fix:** re-enable Real-Time Protection and confirm the compliance state updates on next check-in.

## Lessons Learned

- **Test a shared-cause theory against a device that *doesn't* exhibit the symptom before investigating each affected device individually.** One clean comparison (same OS/build, different outcome) can save significant time versus three parallel investigations
- The instinct to look for one shared explanation is usually right — but confirming it costs very little (one comparison device) relative to investigating several devices in parallel as if they're the same problem
- **"Noncompliant" in Intune can mean very different things — a real, current policy violation; a device that's stale/decommissioned; or a device that's actively used but has stopped checking in.** Each needs a different response, and lumping them together as "compliance issues" obscures which is which
- **A device silently failing to sync for months while still being actively used is arguably the highest-risk state of the three** — it looks normal to the end user and to anyone glancing at it in person, and only shows up by checking last-sync/last-checkin timestamps rather than assuming active use implies active management
- **Periodically audit last-checkin times across the Intune fleet (not just react to compliance-flag alerts)** — a device can drift into this state with zero visible symptoms for the user, and zero alert unless something specifically checks for silence rather than for an active policy failure

## Quick Reference Checklist

| If you see… | Do this |
|---|---|
| Several devices noncompliant at once with an obvious common factor | Check a compliant device with the same factor (same OS/build) before investigating each one |
| Comparison device with the same OS/build is compliant | Drop the shared-cause theory; triage each device separately |
| Device in daily use but last sync months ago | Operational blind spot — not receiving profiles or policy; handle separately from policy failures |
| Device not in use, old last check-in | Offboarding / cleanup |
| Defender Real-Time Protection disabled | Re-enable, confirm compliance updates on next check-in |
