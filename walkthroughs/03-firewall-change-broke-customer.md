# The Firewall Change That Broke a Customer We Didn't Touch

**Stack:** on-prem/hosted Hyper-V, VPN (split-tunnel), DNS, network security policy

## TL;DR

A security hardening project restricted several internally-hosted servers to a static-IP allowlist plus VPN, for anyone else. Applying it to one particular server broke access completely — for internal staff *and* for something we hadn't accounted for at all: an external product-licensing endpoint used by paying customers with no VPN access whatsoever. Two separate problems turned out to be tangled together — a VPN routing failure, and a shared IP address serving two logically unrelated purposes. Untangling them was most of the work; the actual firewall change was rolled back in minutes.

## Background

The security initiative: move a set of internally-hosted servers behind a restrictive firewall policy — only pre-approved static IPs get direct access, everyone else has to come in over VPN. Straightforward, and it had already gone smoothly on two other servers with no complaints.

The third server in scope hosted an internal CRM/licensing application. Applying the same policy to it broke access outright — including from an office IP that was on the approved allowlist, and including over the VPN that was supposed to be the fallback path for everyone else.

## First problem: the VPN itself didn't reach this server

Disconnecting the VPN client restored access instantly. That's the tell for a routing problem, not a firewall-rule problem — if the firewall itself were the blocker, VPN or no VPN wouldn't matter.

The VPN in use was a split-tunnel configuration: only traffic destined for specific internal ranges gets routed through the tunnel, everything else goes out the client's normal internet connection directly. The server in question sat on the hosting provider's public-IP address space (not a private/internal range), so its traffic was being routed into the tunnel's internal gateway and handed off within the hosting provider's own backbone network — never actually reaching the public internet path the server needed, and with nothing in the VPN's own connection logs (audit-only, not per-session) to show *why* it was failing. The server had no private-network interface configured at all, which ruled out a "just route it to the private side" fix without a bigger network change on the hosting provider's end.

That alone was enough reason to roll the firewall change back on this server while a proper fix (either an inbound NAT to a whitelistable address, or a backbone-to-VLAN bridge) got sorted out with the hosting provider.

## Second, bigger problem: a shared IP nobody had flagged

While rolling back, a second and more serious issue turned up: the hostname used for the internal CRM system's web UI, and the hostname used as the OAuth/licensing callback for the company's own commercial product, resolved to **the same IP address** — because they were, in fact, the same physical server.

That licensing endpoint is used by external, paying customers to activate and license their installations of the product. Those customers have no VPN, no static-IP allowlisting arrangement, and no reason to expect one — it's supposed to be a public endpoint. If the firewall restriction had stuck, it wouldn't just have blocked internal CRM access; it would have broken product licensing for every external customer trying to activate or re-validate their installation.

This wasn't visible from the original project scope, which had (reasonably) assumed each server in scope was purely internal-facing. Nobody had mapped which public hostnames pointed at which physical hosts before starting.

## Resolution

- Rolled the firewall policy off this server immediately.
- Filed a support request with the hosting provider describing the VPN routing symptom and asking whether the fix was an inbound NAT rule or a network bridge.
- Flagged the server as a poor candidate for this restriction *as currently scoped* — it can't be treated as internal-only until either the external licensing traffic is moved to a different host, or a way is found to keep that specific path open while still restricting the rest.
- The other two (genuinely internal-only) servers in the project stayed on the restrictive policy with no issues; a fourth internal-only server was left for a later phase, unaffected by any of this.

## Takeaways

- **"Disconnect the VPN and it works" is a routing symptom, not a firewall symptom** — check split-tunnel routes and where traffic to the target's IP range actually goes before assuming a rule is wrong.
- **Before restricting network access to a server, map every public hostname that resolves to it — not just the ones the project remembers it's used for.** A server can be "internal" for one purpose and simultaneously load-bearing for something completely external.
- **A successful rollout on similar-looking servers doesn't predict the next one will go the same way.** Two servers in this project went fine; the third had a hidden dependency the first two didn't.
- **Audit-only VPN/firewall logs (connection established/torn down) won't show you *why* traffic silently drops** — you may need a packet-level trace or a support ticket with the network provider rather than expecting the answer to be in your own logs.
