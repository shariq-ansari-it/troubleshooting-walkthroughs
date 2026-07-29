# Migrating a Golden-Copy VHD Into Azure Across Regions

**Stack:** Azure Managed Disks, Azure Blob Storage, Azure CLI, NSG rules

## TL;DR

Turning a large (~127GB) Hyper-V VHD "golden copy" into a repeatable, disposable Azure test VM hit two separate Azure disk-import restrictions back to back: importing directly from a SAS-token blob URL is flatly rejected, and importing from a storage account via RBAC requires the source storage account to be in the **same region** as the target managed disk — which it wasn't. Fixed with a server-side staged blob copy into a same-region storage account first. A side investigation into what looked like duplicate/conflicting NSG rule entries turned out to be two genuinely different IP addresses from the same office, just easy to misread at a glance.

## Goal

A team needed a repeatable way to spin up a disposable Azure VM from a known-good Hyper-V VHD image, test against it, and tear it down — without repeating a slow, error-prone manual setup each time.

## Attempt 1: import directly from a SAS URL

The first, most direct-seeming approach — pass a SAS-token URL for the blob straight to disk creation:

```bash
az disk create --resource-group <rg> --name <disk-name> --source "<blob-url-with-sas>"
```

This was rejected outright. Azure's managed-disk import from a `--source` URL doesn't support a SAS-token blob URL as a source at all — this isn't a permissions or token-scope problem to fix, it's simply not an accepted input shape for that parameter. No amount of adjusting the SAS token's permissions or expiry changes this.

## Attempt 2: import via storage account RBAC — blocked by region mismatch

The correct mechanism is importing via the source storage account's resource ID (using RBAC/data-plane permissions rather than a SAS token):

```bash
az disk create --resource-group <rg> --name <disk-name> \
  --source-storage-account-id <source-storage-account-resource-id> \
  --location <disk-region>
```

This failed too — because Azure's disk-import-from-storage-account path requires the **source storage account and the target managed disk to be in the same Azure region.** The golden-copy VHD lived in a storage account in one region; the target test environment (VNet, subnet, NSGs already built for it) was in a different region entirely. This is a hard platform restriction, not a quota or permission issue.

## The fix: a staged, same-region copy

Rather than relocating the entire test environment to match the golden copy's region (a much bigger change), the fix was a **server-side blob copy** of the VHD into a new storage account already in the target region:

```bash
az storage blob copy start \
  --account-name <staging-account-in-target-region> \
  --destination-container <container> \
  --destination-blob <name>.vhd \
  --source-uri "<source-blob-url-with-sas>"
```

Server-side copy means the data moves directly between Azure storage accounts without round-tripping through a local machine — important given the file size. Once the copy completed, `az disk create --source-storage-account-id` against the *staged* (same-region) storage account worked immediately.

The staged copy was deliberately left in place afterward rather than deleted, since the same cross-region copy step will be needed again for the next version of the golden image — reusing the staging location skips repeating the (slow, large) copy from scratch.

## A smaller side-investigation: "duplicate" NSG entries that weren't

While reviewing network security group rules for the test subnet, two entries that looked like they might be accidental duplicates for the same person turned out, on closer inspection, to be genuinely different IP addresses — one from a colleague's WiFi egress and one from the same office's separate LAN egress path, both legitimately needing access from the same physical location but exiting through different public IPs depending on connection type. A reminder that "these two entries look suspiciously similar" is worth actually diffing character-by-character before assuming it's a mistake and removing one.

## Takeaways

- **Azure managed-disk import from a URL does not accept a SAS-token blob URL as a source** — use the storage-account-resource-ID path (RBAC-based) instead; don't spend time adjusting SAS token scope or expiry expecting that to fix it.
- **Disk import via storage-account resource ID requires the source storage account and destination disk to be in the same region** — check this before attempting the import, not after the first rejection, since it changes the whole approach (staged copy vs. direct import).
- **A server-side blob copy (`az storage blob copy start`) moves data directly between storage accounts without transiting a local machine** — the right tool for moving large images between regions rather than downloading and re-uploading manually.
- **Keep a same-region staged copy around if the source image is reused periodically** — it turns a slow one-time cross-region transfer into a one-time cost instead of a recurring one.
- **Two nearly-identical-looking NSG rule entries aren't automatically a duplicate mistake** — diff the actual values before removing either one; the same person/location can legitimately need multiple entries for different egress paths.
