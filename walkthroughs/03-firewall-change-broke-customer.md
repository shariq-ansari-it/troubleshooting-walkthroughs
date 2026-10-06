# The Firewall Change That Broke a Customer We Didn't Touch

## TL;DR

A security hardening project restricted several internally-hosted servers to a static-IP allowlist plus VPN, for anyone else. Applying it to one particular server broke access completely — for internal staff *and* for something we hadn't accounted for at all: an external product-licensing endpoint used by paying customers with no VPN access whatsoever. Two separate problems turned out to be tangled together — a VPN routing failure, and a shared IP address serving two logically unrelated purposes. Untangling them was most of the work; the actual firewall change was rolled back in minutes.

## Purpose of this Document

A case study and reference runbook for the problem summarised in the TL;DR above.

It is intentionally written to:

- Record the response that was actually taken (rollback, provider ticket, re-scoping)
- Separate the two independent root causes so each can be recognised on its own
- Give a pre-change check to stop the same thing happening on the next server

## Environment

- On-prem/hosted Hyper-V servers on the hosting provider's public-IP address space
- Firewall / network security policy: approved static IPs direct, everyone else via VPN
- Split-tunnel VPN (connection logs are audit-only, not per-session)
- DNS — multiple public hostnames resolving to the same server
- Affected server: internal CRM/licensing application, no private-network interface configured

---

## Correct End-to-End Process (Authoritative)

The policy had already gone smoothly on two other servers. On the third, access broke outright — including from an office IP on the approved allowlist, and including over the VPN that was meant to be the fallback for everyone else.

### 1. Distinguish Routing From Firewall

Disconnect the VPN client and retest. Here it restored access instantly — the tell for a routing problem, not a firewall-rule problem. If the firewall were the blocker, VPN or no VPN wouldn't matter. See [Issue 1](#issue-1--vpn-doesnt-reach-the-server).

### 2. Roll the Firewall Policy Off the Server

Roll it back immediately. The VPN routing failure alone justifies this while a proper fix is arranged.

### 3. Map Every Hostname Resolving to the Server

While rolling back, check which public hostnames resolve to this server's IP — not just the ones the project remembers it's used for. Here the internal CRM web UI hostname and the product's OAuth/licensing callback hostname resolved to **the same IP address**. See [Issue 2](#issue-2--shared-ip-serving-an-external-licensing-endpoint).

### 4. Raise the Routing Issue With the Hosting Provider

File a support request describing the VPN routing symptom and asking whether the fix is:

- an inbound NAT to a whitelistable address, or
- a backbone-to-VLAN bridge

### 5. Re-scope the Project

- Flag the server as a poor candidate for this restriction *as currently scoped* — it can't be treated as internal-only until either the external licensing traffic moves to a different host, or a way is found to keep that specific path open while restricting the rest.
- Leave the two genuinely internal-only servers on the restrictive policy (no issues).
- A fourth internal-only server stays in a later phase, unaffected.

**If this all happens → internal and customer access restored, and the restriction stays on the servers it's safe for.**

---

## Issues Encountered in This Case

### Issue 1 — VPN doesn't reach the server

**Observed:** no access over VPN; disconnecting the VPN client restored access instantly. Nothing in the VPN's own connection logs explained it.

**Root cause:** split-tunnel VPN — only traffic for specific internal ranges goes through the tunnel. The server sat on the hosting provider's public-IP space (not a private range), so its traffic was routed into the tunnel's internal gateway and handed off within the provider's own backbone, never reaching the public internet path the server needed. The VPN logs are audit-only (connection established/torn down), not per-session, so they don't show *why* traffic drops.

**What didn't work:** a "just route it to the private side" fix — the server has no private-network interface, so that needs a bigger network change on the provider's end.

**Fix:** rollback on this server; provider ticket for inbound NAT or backbone-to-VLAN bridge ([step 4](#4-raise-the-routing-issue-with-the-hosting-provider)).

### Issue 2 — Shared IP serving an external licensing endpoint

**Observed:** the CRM web UI hostname and the product's OAuth/licensing callback hostname resolved to the same IP — they were the same physical server.

**Root cause:** the licensing endpoint is used by external, paying customers to activate and license their installations. They have no VPN, no static-IP allowlisting, and no reason to expect one — it's meant to be public. Had the restriction stuck, it would have broken product licensing for every external customer activating or re-validating. The project scope had (reasonably) assumed every in-scope server was purely internal-facing; nobody had mapped public hostnames to physical hosts before starting.

**Fix:** keep the restriction off this server until the licensing traffic is moved or a way is found to keep that path open ([step 5](#5-re-scope-the-project)).

## Lessons Learned

- **"Disconnect the VPN and it works" is a routing symptom, not a firewall symptom** — check split-tunnel routes and where traffic to the target's IP range actually goes before assuming a rule is wrong.
- **Before restricting network access to a server, map every public hostname that resolves to it — not just the ones the project remembers it's used for.** A server can be "internal" for one purpose and simultaneously load-bearing for something completely external.
- **A successful rollout on similar-looking servers doesn't predict the next one will go the same way.** Two servers in this project went fine; the third had a hidden dependency the first two didn't.
- **Audit-only VPN/firewall logs (connection established/torn down) won't show you *why* traffic silently drops** — you may need a packet-level trace or a support ticket with the network provider rather than expecting the answer to be in your own logs.

## Quick Reference Checklist

| If you see… | Do this |
|---|---|
| Access fails on VPN, works with VPN disconnected | Routing, not firewall — check split-tunnel routes for the server's IP range |
| Allowlisted IP also blocked after the change | Roll back first, then investigate |
| Server on provider public-IP space, no private interface | Ask provider for inbound NAT or backbone-to-VLAN bridge |
| VPN logs show only connect/disconnect | Packet-level trace or provider ticket — the answer won't be in your logs |
| About to restrict a server | Map every public hostname resolving to its IP first |
| A hostname on the server serves external customers | Don't treat the server as internal-only |
