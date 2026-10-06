# An RDS Environment's Performance Collapse, Triggered by Onboarding

## Purpose of this Document

A case study and runbook for a hosted Remote Desktop Services environment that slowed down right after several new users were onboarded. The new users weren't the direct cause — they were the trigger that pushed pre-existing, compounding resource pressures over a threshold at the same time: a SQL Server transaction log consuming nearly all free disk space, RAM pressure from the extra sessions, and a desktop-OS service running needlessly on a server host. This document owns SQL transaction log growth diagnosis and fix.

It is intentionally written to:

- Give the diagnosis checks and the fix order that worked
- Separate the obvious-but-incomplete theory (new users = too much load) from the actual causes
- Record each contributing cause with its symptom, cause and fix
- Act as the reference for diagnosing and fixing SQL transaction log growth safely

## Environment

- Windows Remote Desktop Services (RDS) host, multi-user
- Line-of-business application (Sage-based) on the RDS host
- SQL Server (application database)
- Windows services / resource management
- Windows Update (queued)

---

## Correct End-to-End Process (Authoritative)

### 1. Don't Stop at "The New Users Are Too Much Load"

A slowdown that coincides with new users is easy to put down to session load alone. That's part of the picture, but check for pre-existing problems the extra load has exposed — see [Issue 1](#issue-1--slowdown-blamed-on-the-new-users).

### 2. Check Free Disk Space

Check free disk space on the RDS host. Here it was critically low — roughly 6.5GB free — caused by SQL Server transaction log growth. See [Issue 2](#issue-2--sql-transaction-log-consuming-the-disk).

### 3. Check Memory

Check RAM utilization against what it had been running at before onboarding. See [Issue 3](#issue-3--memory-pressure-from-additional-sessions).

### 4. Audit Running Services

Look for desktop-oriented services that serve no purpose on a server/RDS host — here, the **Windows Push Notification** service. See [Issue 4](#issue-4--windows-push-notification-service-running-on-a-server).

### 5. Fix in Priority Order

Disk space was the most acute — and the easiest to make worse by doing things in the wrong order.

1. **SQL transaction log cleanup** first — freeing critical disk space immediately, since everything else on the host (including Windows Update, which was also queued) was constrained by how little free space remained. Find out *why* the log is growing before shrinking anything:

   ```sql
   SELECT name, recovery_model_desc, log_reuse_wait_desc FROM sys.databases;
   DBCC SQLPERF(LOGSPACE);
   ```

   `FULL` recovery with no log backups is the usual cause — either schedule log backups or switch to `SIMPLE` if point-in-time recovery isn't needed, then shrink the log through SQL. **Never delete `.ldf` files to "clear SQL logs"** — that can take the database offline. Old log backups and SQL `ERRORLOG` files are the only things safe to delete by hand.
2. **Disable the unnecessary Windows Push Notification service** — a small but free resource-consumption win with genuinely zero downside on a server host.
3. **Schedule a Windows Update maintenance window** — deferred to a planned time rather than run immediately, since the environment was already under pressure and an update cycle mid-incident would have added more load, not relieved it.

---

## Issues Encountered in This Case

### Issue 1 — Slowdown blamed on the new users

**Observed:** an RDS environment used for a multi-user, Sage-based line-of-business application, previously running acceptably, suddenly felt sluggish across the board — slow logins, slow application responsiveness — starting right around when a small batch of new users were added.

**What didn't work:** treating the new session load as the *whole* picture would have missed two other real problems already present before the new users arrived — they just hadn't yet pushed things far enough to be user-visible.

**Root cause:** the new users were the load increment that exposed pre-existing, compounding problems ([Issue 2](#issue-2--sql-transaction-log-consuming-the-disk), [Issue 3](#issue-3--memory-pressure-from-additional-sessions), [Issue 4](#issue-4--windows-push-notification-service-running-on-a-server)).

### Issue 2 — SQL transaction log consuming the disk

**Observed:** free disk space on the RDS host critically low — roughly 6.5GB free.

**Root cause:** SQL Server's transaction log for the application database had grown substantially over time without regular truncation/maintenance, quietly consuming disk space in the background long before it became critical. Low free disk space affects far more than "running out of room" — it degrades virtual memory paging, SQL Server's own operation and general OS responsiveness well before the disk is actually full.

**Fix:** diagnose and shrink the log through SQL — see [step 5](#5-fix-in-priority-order), item 1. Never delete `.ldf` files.

### Issue 3 — Memory pressure from additional sessions

**Observed:** RAM utilization meaningfully higher than it had been running.

**Root cause:** the additional concurrent user sessions from onboarding. Not a single root cause on its own, but a compounding factor sitting on top of the disk-space problem.

### Issue 4 — Windows Push Notification service running on a server

**Observed:** the **Windows Push Notification** service running on the RDS host.

**Root cause:** a component with a legitimate purpose on a normal desktop OS (delivering app/system notifications to an interactively used PC) but **no purpose at all on a server acting as an RDS host** serving remote sessions. A holdover from the base OS image rather than something anyone had deliberately enabled, consuming resources for a function nobody needed.

**Fix:** disable the service — zero downside on a server host.

## Lessons Learned

- **A slowdown that coincides with new user onboarding isn't necessarily *caused* by the new users** — they can just as easily be the load increment that finally exposes pre-existing, compounding problems (disk space, log growth, unnecessary services) that had been quietly building
- **SQL Server transaction log growth is a classic silent disk-space consumer** — check it specifically, and have regular log maintenance/truncation in place, rather than discovering it only once free space becomes critical
- **Server hosts built from general-purpose OS images can carry desktop-oriented services that serve no purpose in a server role** (notification services being a clear example) — audit what's actually running on an RDS/server host versus what a default image includes
- **When multiple problems are found during one incident, sequence the fixes by urgency and by what they unblock** — freeing disk space first wasn't just "the biggest problem", it was also a prerequisite for other maintenance (like Windows Update) to run safely at all

## Quick Reference Checklist

| If you see… | Do this |
|---|---|
| RDS slowdown right after onboarding users | Check disk, RAM and running services — don't stop at session load |
| Critically low free disk on a host running SQL Server | Check transaction log: `sys.databases` (`recovery_model_desc`, `log_reuse_wait_desc`) and `DBCC SQLPERF(LOGSPACE)` |
| `FULL` recovery with no log backups | Schedule log backups, or switch to `SIMPLE` if point-in-time recovery isn't needed; then shrink the log through SQL |
| Tempted to delete `.ldf` files | Don't — can take the database offline; only old log backups and `ERRORLOG` files are safe to delete by hand |
| Windows Push Notification service running on an RDS host | Disable it |
| Windows Update queued during the incident | Free disk first; schedule updates in a maintenance window, not mid-incident |
