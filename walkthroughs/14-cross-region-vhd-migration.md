# Migrating a Golden-Copy VHD Into Azure Across Regions

## Purpose of this Document

A case study and runbook for turning a large (~127GB) Hyper-V VHD "golden copy" into a repeatable, disposable Azure test VM — spin up from a known-good image, test, tear down — without repeating a slow, error-prone manual setup each time. The import hit two Azure disk-import restrictions back to back (no SAS-URL source; source storage account must be in the same region as the disk), fixed with a server-side staged blob copy into a same-region storage account.

It is intentionally written to:

- Give the working import path, in order
- Keep it separate from the two rejected approaches, so they aren't retried
- Record a side investigation into NSG entries that looked like duplicates but weren't
- Make the next golden-image version a repeat, not a rediscovery

## Environment

- Hyper-V VHD golden copy (~127GB)
- Azure Blob Storage (source account in one region; staging account in the target region)
- Azure Managed Disks
- Azure CLI
- Target test environment: VNet, subnet and NSG rules already built, in a different region from the source

---

## Correct End-to-End Process (Authoritative)

### 1. Check the Regions First

Disk import via storage-account resource ID requires the **source storage account and the target managed disk to be in the same Azure region**. Check this before attempting the import — it decides the whole approach (staged copy vs. direct import). Don't try a SAS URL as the `--source`; it isn't accepted. See [Issue 1](#issue-1--import-directly-from-a-sas-url-rejected) and [Issue 2](#issue-2--import-via-storage-account-rbac-blocked-by-region-mismatch).

### 2. Stage the VHD Into a Same-Region Storage Account

If the regions differ, do a **server-side blob copy** into a storage account already in the target region, rather than relocating the whole test environment to the golden copy's region (a much bigger change):

```bash
az storage blob copy start \
  --account-name <staging-account-in-target-region> \
  --destination-container <container> \
  --destination-blob <name>.vhd \
  --source-uri "<source-blob-url-with-sas>"
```

Server-side copy moves the data directly between Azure storage accounts without round-tripping through a local machine — important given the file size.

### 3. Create the Managed Disk From the Staged Copy

Once the copy completes, import via the **staged** (same-region) storage account's resource ID:

```bash
az disk create --resource-group <rg> --name <disk-name> \
  --source-storage-account-id <source-storage-account-resource-id> \
  --location <disk-region>
```

Expected: works immediately against the staged account.

### 4. Keep the Staged Copy

Leave the staged copy in place. The same cross-region copy step will be needed for the next version of the golden image — reusing the staging location skips repeating the slow, large copy from scratch.

### 5. Review NSG Rules Carefully

When reviewing NSG rules for the test subnet, diff near-identical entries character by character before removing either. See [Issue 3](#issue-3--duplicate-nsg-entries-that-werent).

**If this all happens → managed disk created in the target region, ready for the disposable test VM.**

---

## Issues Encountered in This Case

### Issue 1 — Import directly from a SAS URL rejected

**Observed:** passing a SAS-token URL for the blob straight to disk creation was rejected outright:

```bash
az disk create --resource-group <rg> --name <disk-name> --source "<blob-url-with-sas>"
```

**Root cause:** Azure's managed-disk import from a `--source` URL doesn't support a SAS-token blob URL as a source at all. It isn't a permissions or token-scope problem — it's simply not an accepted input shape for that parameter.

**What didn't work:** adjusting the SAS token's permissions or expiry changes nothing.

**Fix:** use the storage-account resource ID path (RBAC/data-plane permissions rather than a SAS token) — `--source-storage-account-id`.

### Issue 2 — Import via storage account RBAC blocked by region mismatch

**Observed:** the correct mechanism also failed:

```bash
az disk create --resource-group <rg> --name <disk-name> \
  --source-storage-account-id <source-storage-account-resource-id> \
  --location <disk-region>
```

**Root cause:** disk import from a storage account requires the **source storage account and the target managed disk to be in the same Azure region.** The golden-copy VHD lived in a storage account in one region; the target test environment (VNet, subnet, NSGs already built) was in a different region. A hard platform restriction, not a quota or permission issue.

**Fix:** server-side staged copy into a same-region storage account ([step 2](#2-stage-the-vhd-into-a-same-region-storage-account)), then import from that.

### Issue 3 — "Duplicate" NSG entries that weren't

**Observed:** two NSG rule entries for the test subnet looked like accidental duplicates for the same person.

**Root cause:** on closer inspection, genuinely different IP addresses — one from a colleague's WiFi egress and one from the same office's separate LAN egress path. Both legitimately needed access from the same physical location but exited through different public IPs depending on connection type.

**Fix:** none needed — kept both. Diff suspiciously similar entries character by character before assuming a mistake.

## Lessons Learned

- **Azure managed-disk import from a URL does not accept a SAS-token blob URL as a source** — use the storage-account resource ID path (RBAC-based) instead; don't spend time adjusting SAS token scope or expiry expecting that to fix it
- **Disk import via storage-account resource ID requires the source storage account and destination disk to be in the same region** — check this before attempting the import, not after the first rejection, since it changes the whole approach (staged copy vs. direct import)
- **A server-side blob copy (`az storage blob copy start`) moves data directly between storage accounts without transiting a local machine** — the right tool for moving large images between regions rather than downloading and re-uploading manually
- **Keep a same-region staged copy around if the source image is reused periodically** — it turns a slow cross-region transfer into a one-time cost instead of a recurring one
- **Two nearly identical-looking NSG rule entries aren't automatically a duplicate mistake** — diff the actual values before removing either; the same person/location can legitimately need multiple entries for different egress paths

## Quick Reference Checklist

| If you see… | Do this |
|---|---|
| `az disk create --source "<blob-url-with-sas>"` rejected | SAS URL isn't an accepted source — use `--source-storage-account-id` |
| `--source-storage-account-id` import fails, source in another region | `az storage blob copy start` into a storage account in the disk's region, import from that |
| Next golden-image version to import | Reuse the existing same-region staging account |
| Two near-identical NSG entries for the same person/location | Diff the IPs — may be separate WiFi and LAN egress |
