# An RDS Environment's Performance Collapse, Triggered by Onboarding

**Stack:** Windows Remote Desktop Services, SQL Server, Windows services/resource management

## TL;DR

A hosted Remote Desktop Services environment started running noticeably slower right after several new users were onboarded onto it. The new users themselves weren't the direct cause — they were the trigger that pushed several pre-existing, compounding resource pressures over a threshold at the same time: a SQL Server transaction log that had grown to consume nearly all remaining free disk space, general RAM pressure from the additional concurrent sessions, and an entirely unrelated desktop-OS service running needlessly on a server host and consuming resources for no purpose at all.

## The symptom

An RDS environment used for a line-of-business application (Sage-based, multi-user) that had been running acceptably suddenly felt sluggish across the board — slow logins, slow application responsiveness — starting right around when a small batch of new users were added to it.

## Investigation

The instinct when a slowdown coincides with new users is to assume the new session load itself is simply too much for the environment. That's part of the picture, but treating it as the *whole* picture would have missed two other real problems already present before the new users arrived — they just hadn't yet pushed things far enough to be user-visible.

**Disk space.** A check of free disk space on the RDS host found it critically low — down to roughly 6.5GB free. Root cause: SQL Server's transaction log for the application database had grown substantially over time without regular truncation/maintenance, quietly consuming disk space in the background for a long time before it became critical. Low free disk space affects far more than just "running out of room" — it degrades virtual memory paging performance, SQL Server's own operation, and general OS responsiveness well before the disk is actually full.

**Memory pressure.** The additional concurrent user sessions from onboarding pushed RAM utilization meaningfully higher than it had been running. Not a single root cause on its own, but a compounding factor sitting on top of the disk-space problem.

**A rogue service.** Also found running: the **Windows Push Notification** service — a component with a legitimate purpose on a normal desktop OS (delivering app/system notifications to an interactively-used PC) but with **no purpose at all on a server acting as an RDS host** serving remote sessions. It was running anyway, consuming resources for a function nobody needed in that context — a holdover from the base OS image rather than something anyone had deliberately enabled.

## Resolution

Addressed in priority order, since disk space was the most acute (and easiest to make worse by doing things in the wrong order):

1. **SQL transaction log cleanup** first — freeing critical disk space immediately, since everything else on the host (including Windows Update, which was also queued) was constrained by how little free space remained.
2. **Disable the unnecessary Windows Push Notification service** — a small but free resource-consumption win with genuinely zero downside on a server host.
3. **Schedule a Windows Update maintenance window** — deferred to a planned time rather than run immediately, since the environment was already under pressure and an update cycle mid-incident would have added more load, not relieved it.

## Takeaways

- **A slowdown that coincides with new user onboarding isn't necessarily *caused* by the new users** — they can just as easily be the load increment that finally exposes pre-existing, compounding problems (disk space, log growth, unnecessary services) that had been quietly building for a while.
- **SQL Server transaction log growth is a classic silent disk-space consumer** — worth checking specifically, and worth having regular log maintenance/truncation in place, rather than discovering it only once free space becomes critical.
- **Server hosts built from general-purpose OS images can carry desktop-oriented services that serve no purpose in a server role** (notification services being a clear example) — worth an audit of what's actually running on an RDS/server host versus what a default image includes.
- **When multiple problems are found during one incident, sequence the fixes by urgency and by what they unblock** — freeing disk space first here wasn't just "the biggest problem," it was also a prerequisite for other maintenance (like Windows Update) to run safely at all.
