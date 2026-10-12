# A Second-Screen Azure Dashboard Instead of a Heavy Portal Tab

## TL;DR

Safari had replaced a Chromium browser as the daily driver on an Apple Silicon Mac, on the assumption that a native browser would use less memory and CPU. It didn't feel that way. Measuring showed the browser engine using 6.6 GB across 21 page processes, but memory pressure was low (82% free, almost no swapping on a 32 GB machine). The browser was not the problem. Two pages were: an Azure portal web app left open for hours on a detail page (about 2.9 GB) and a long-running video tab (about 1.5 GB). Closing those two took the total to 2.3 GB.

The portal was open because it was being used as a monitoring screen. It was replaced by a shared Azure portal dashboard built entirely from Resource Graph tiles across all subscriptions: VM power state, AVD session host health, Arc agent status, fired alerts, patch runs, pending patches and service health. It's generated from a script, deployed with one `az rest` call, and costs nothing to run. The first build surfaced four live problems nobody had been watching for.

## Purpose of this Document

A case study and reference runbook for the problem summarised in the TL;DR above.

It is intentionally written to:

- Show how to measure what a browser on macOS is really costing before switching browsers
- Give a complete, repeatable way to build and deploy a Resource Graph dashboard from code
- Record the deployment and query errors hit on the way, with the exact fixes
- Explain how to keep a portal dashboard open all day without its memory creeping up

## Environment

- Apple Silicon MacBook Pro (M1 Max, 32 GB), macOS 27, Safari with portal pages installed as Safari web apps
- Azure tenant with several subscriptions: Azure VMs, Azure Virtual Desktop host pools, Arc-enabled on-prem servers, Azure Update Manager, Azure Monitor alerts
- Azure CLI with the `resource-graph` extension, Python 3 for the generator script
- Browser constraint: no Chrome or Edge (privacy), so staying on Safari was a requirement, not just a preference

---

## Correct End-to-End Process (Authoritative)

### 1. Measure Before Blaming the Browser

Every Safari tab and every Safari web app runs in its own `com.apple.WebKit.WebContent` process. Total them and check system memory pressure in the same pass:

```bash
# Number of page processes, total memory and CPU for everything WebKit
ps -axo comm | grep -c '[W]ebKit.WebContent'
ps -axo rss,pcpu,comm | grep '[W]ebKit' \
  | awk '{r+=$1; c+=$2} END {printf "WebKit total: %.1f GB, %.1f%% CPU\n", r/1048576, c}'

# Is the machine actually short of memory?
memory_pressure | tail -1          # "System-wide memory free percentage: NN%"
vm_stat | grep Pageouts            # low number over days of uptime = not swapping

# Largest page processes and how long they've been running
ps -axo rss,etime,comm | grep '[W]ebKit.WebContent' | sort -rn | head -5 \
  | awk '{printf "%5d MB  up %s\n", $1/1024, $2}'
```

A high WebKit total with a high free percentage and few pageouts means memory is being used, not exhausted. Don't change browsers on that evidence — see [Issue 1](#issue-1--the-native-browser-felt-heavy).

### 2. Find the Expensive Pages

List what's open, then match the largest processes to pages by closing candidates and measuring again.

```bash
# Safari tab domains (read-only AppleScript)
osascript -e 'tell application "Safari" to get URL of every tab of every window' \
  | tr ',' '\n' | sed -E 's#^ ?(https?://[^/]+).*#\1#'

# Which Safari web apps are actually running (not just installed)
ps -axo pid,rss,etime,command | grep '[W]eb App.app'
osascript -e 'tell application "System Events" to tell process "Web App" to get name of every window'
```

Installed web apps that aren't running cost nothing — see [Issue 2](#issue-2--installed-web-apps-counted-as-running).

Measured here:

| State | Page processes | WebKit memory | WebKit CPU | System free |
|---|---|---|---|---|
| Start | 21 | 6.6 GB | 16% | 82% |
| Portal web app closed (open about 5 hours on a detail page) | 13 | 3.7 GB | 9% | 84% |
| Long-running video and forum tabs closed | 13 | 2.3 GB | 0.6% | 85% |

### 3. Decide What the Monitoring Screen Needs

The portal was kept open on a second monitor to watch the estate. List the questions it was answering, and make each one a tile:

| Question | Tile | Resource Graph table |
|---|---|---|
| How many VMs are on? | VMs running (single value) and a list | `resources` |
| Can users get to their desktops? | AVD hosts available / unavailable, hosts by status, powered-on host list | `desktopvirtualizationresources` |
| Are the on-prem servers still reporting? | Arc servers not connected, Arc status list | `resources` (`microsoft.hybridcompute/machines`) |
| Has anything alerted? | Alerts fired in the last 24 hours | `alertsmanagementresources` |
| Did patching run? | Patch runs (30 days), pending patches by machine | `patchinstallationresources`, `patchassessmentresources` |
| Is Azure itself having a problem? | Active service health events | `servicehealthresources` |

Resource Graph tiles query the control plane directly, with no Log Analytics workspace, no agents and no cost. They show only what the viewer has read access to, the same RBAC filtering described in [walkthrough 05](05-vm-patch-window-automation.md) (fewer rows, not an error).

### 4. Test Every Query From the CLI First

A broken query in a dashboard tile just shows an error in the tile. Run each one with `az graph query` before building anything:

```bash
az graph query -q "desktopvirtualizationresources
  | where type =~ 'microsoft.desktopvirtualization/hostpools/sessionhosts'
  | extend Pool=tostring(split(name,'/')[0]), Host=tostring(split(name,'/')[1]),
           Status=tostring(properties.status), Sessions=toint(properties.sessions),
           Heartbeat=todatetime(properties.lastHeartBeat)
  | project Pool, Host, Status, Sessions, Heartbeat" --query data -o table
```

Use `--query data -o table`. Plain `-o table` prints only the record count. Two query issues came up here: [Issue 4](#issue-4--parserfailure-on-propertiestitle) (a reserved word) and [Issue 5](#issue-5--service-health-alerts-never-leave-fired) (alerts that never resolve).

### 5. Generate the Dashboard JSON From a Script

The generator ([Appendix A](#appendix-a--dashboard-generator-script)) defines every tile and its position in one Python file. It writes the ARM JSON and, with `--queries`, prints each tile query so step 4 can be repeated automatically:

```bash
python3 Build-OpsDashboard.py --location uksouth --out ops-live.json
python3 Build-OpsDashboard.py --queries | while IFS= read -r q; do
  az graph query -q "$q" --first 5 --query "length(data)" -o tsv
done
```

Every line should print a row count, not an error. Zero is fine for service health when nothing is active.

The tile types and their JSON shape come from Microsoft's Resource Graph dashboard samples (`Azure-Samples/Governance`, `src/resource-graph/portal-dashboards`). Download one and inspect it rather than guessing:

| Part type (`Extension/HubsExtension/PartType/...`) | Use | Inputs |
|---|---|---|
| `MarkdownPart` | Header, links | `settings.content.settings.content` (HTML allowed) |
| `ArgQuerySingleValueTile` | Big number | `partTitle`, `query` returning one row, one column |
| `ArgQueryChartTile` | Chart | `partTitle`, `query`, `chartType` (`1` = bar; see [Issue 6](#issue-6--chart-type-values-are-undocumented)) |
| `ArgQueryGridTile` | Table | `partTitle`, `query` |

Positions use a grid of `x`, `y`, `colSpan`, `rowSpan`. A width of about 20 columns fits a 1440p second monitor without scrolling sideways.

### 6. Deploy With One `az rest` PUT

```bash
az rest --method put \
  --url "https://management.azure.com/subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.Portal/dashboards/ops-live?api-version=2015-08-01-preview" \
  --body @ops-live.json --query name -o tsv
```

The API version matters — see [Issue 3](#issue-3--dashboard-put-rejected-cannot-deserialize-into-list). A dashboard deployed this way is a shared dashboard: anyone with read on the resource group can open it, and each viewer sees only data they can read.

Verify it saved with every tile:

```bash
az rest --method get \
  --url "https://management.azure.com/subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.Portal/dashboards/ops-live?api-version=2020-09-01-preview" \
  --query "length(properties.lenses[0].parts)"
```

(Reading back with the newer API version works and returns arrays, which makes the count easy.)

### 7. Set It Up on the Second Screen

1. Open it: **Portal > Dashboard > dashboard picker > Shared dashboards**.
2. In the dashboard toolbar, set **Auto refresh** to 5 or 10 minutes.
3. Make the window full screen (Ctrl+Cmd+F).
4. Reload once a day (Cmd+R) — see [Ongoing](#ongoing).

**If this all happens → complete.** The second screen shows estate health across all subscriptions from one lightweight page, with the browser kept.

---

## Ongoing

- **Reload daily.** The portal is a single-page app. Every detail page opened from the dashboard (VM, Automation account, logs) stays loaded behind the breadcrumb trail until the page reloads. That growth caused the 2.9 GB seen in step 2, not the dashboard itself.
- **Change tiles in the script, not the portal.** A redeploy replaces the whole dashboard, including layout changes made in **Edit** mode. If a layout tweak in the portal is worth keeping, get the JSON from **Export > Download** in the dashboard toolbar and copy the positions into the script.
- **Use alerts for anything that needs action.** The dashboard is a glance view. Failed Automation jobs, VM availability and budgets belong in Azure Monitor alerts with an action group, so nobody has to watch the screen to catch them.

## Issues Encountered in This Case

### Issue 1 — The "native" browser felt heavy

**Observed:** switching from a Chromium browser to Safari for its supposed efficiency didn't make the machine feel lighter. Activity Monitor showed many large WebKit processes.

**Root cause:** the expectation was wrong, not the browser. Safari's efficiency advantage is mainly energy (battery life on video and light browsing), not memory in heavy JavaScript apps like the Azure portal. A big single-page app costs a lot in any engine. Forum reports of Safari WebContent memory growth on recent macOS exist, but there are no controlled Safari-vs-Chromium memory comparisons on Apple Silicon to back up either claim.

**What didn't help:** looking only at per-process memory. Without memory pressure and pageouts, a high number looks like a problem when it isn't.

**Fix:** measure first ([step 1](#1-measure-before-blaming-the-browser)), close the two expensive pages, and replace the always-open portal page with a lighter dashboard. The browser stayed.

### Issue 2 — Installed web apps counted as running

**Observed:** eight portal web apps were installed (`~/Applications/*.app`), and the first estimate assumed all of them were adding to the memory total.

**Root cause:** only one was running. A Safari web app runs under a single `Web App` process with one window per app, and costs nothing when it isn't open.

**Fix:** check running processes and window names ([step 2](#2-find-the-expensive-pages)), not the installed list.

### Issue 3 — Dashboard PUT rejected: "Cannot deserialize ... into type List"

**Observed:**

```
Bad Request: The request content was invalid and could not be deserialized:
'Cannot deserialize the current JSON object (e.g. {"name":"value"}) into type
'System.Collections.Generic.List`1[...DashboardLensDefinition_V2020_09_01]' ...
Path 'properties.lenses.0'
```

**Root cause:** Microsoft's samples key `lenses` and `parts` by index as objects (`"lenses": {"0": {"parts": {"0": ...}}}`). That is the `2015-08-01-preview` schema. `2020-09-01-preview` expects arrays.

**Fix:** deploy with `api-version=2015-08-01-preview` ([step 6](#6-deploy-with-one-az-rest-put)). The alternative is converting the JSON to arrays and using the newer version. Nothing is created when the PUT fails, so retrying is safe.

### Issue 4 — ParserFailure on `properties.title`

**Observed:** the service health query failed with `ParserFailure` and `token: "."` followed by `token: "title"`.

**Root cause:** `title` is a reserved word in the Resource Graph query language, so dot notation can't reach it.

**Fix:** use bracket notation: `tostring(properties['title'])`.

### Issue 5 — Service health alerts never leave "Fired"

**Observed:** a "fired alerts" tile listed weeks-old service health notifications alongside a real application alert from minutes earlier.

**Root cause:** activity-log alerts (service health, administrative operations) are stateless. They fire once and never move to `Resolved`, so `monitorCondition == 'Fired'` matches them forever.

**Fix:** filter on time as well as state: `where tostring(e.monitorCondition) == 'Fired' and todatetime(e.startDateTime) > ago(24h)`.

### Issue 6 — Chart type values are undocumented

**Observed:** the samples only ever use `chartType: 1` (bar). No public reference for the other values (donut, map) was found.

**Fix:** use `1` in code. To use a donut, change the chart in the portal's **Edit** mode, then export the dashboard to read the value it saved, and put that in the script.

## Results

- Browser engine memory went from 6.6 GB to 2.3 GB, and CPU from 16% to 0.6%, by closing two pages. The browser was kept.
- Dashboard deployed with 14 parts (header plus 13 tiles). All queries pre-tested, saved state verified by reading it back.
- The first build surfaced four live problems:
  - Session hosts whose VMs were running but showed `Unavailable` to AVD
  - Two Arc agents that had gone `Disconnected` that afternoon, plus a third `Expired` for three months
  - No Azure VM in the patch run history at all, only Arc servers (the failure covered in [walkthrough 05](05-vm-patch-window-automation.md))
  - An application error alert fired minutes earlier

## Lessons Learned

- **Measure memory pressure, not memory use.** High use with plenty free and no swapping is a working system. Check `memory_pressure` and pageouts before changing tools.
- **"Native" is not the same as "light."** The cost of a heavy web app comes from the app, and every engine pays it. Choose a browser on other grounds: privacy, extensions, battery.
- **The page left open longest is usually the most expensive one.** Single-page apps grow with every view opened. A daily reload is a cheap fix.
- **A monitoring screen should be a summary, not a live detail page.** Resource Graph tiles answer "is anything wrong?" across every subscription for free, and the detail page is one click away when needed.
- **Build dashboards from code and test queries outside them.** A script plus a CLI pre-flight makes the dashboard reproducible and catches query errors before they show up as broken tiles.
- **Copy API shapes from a working sample, and use that sample's API version.** Portal resource schemas changed between versions without changing the resource type.
- **A new view of old data finds old problems.** Four issues were visible in Resource Graph all along; nobody had put them on one screen.

## Quick Reference Checklist

| If you see… | Do this |
|---|---|
| Browser feels heavy, many large WebContent processes | Check `memory_pressure` and pageouts before blaming the browser |
| One WebContent process far bigger than the rest | It's one page. Find it by closing candidates and re-measuring |
| Portal page using gigabytes after hours open | Reload it (Cmd+R). Use a dashboard instead of a detail page |
| Want a free cross-subscription status screen | Resource Graph tiles in a shared portal dashboard |
| `az graph query -o table` shows only Count/Total_records | Add `--query data` |
| Dashboard PUT: "Cannot deserialize ... into type List" | Use `api-version=2015-08-01-preview` for object-keyed JSON |
| ARG `ParserFailure` near `.title` | Use `properties['title']` |
| Fired alerts tile full of old service health entries | Add a time filter: stateless alerts never resolve |
| Tile shows fewer resources for a colleague | Expected: Resource Graph filters by the viewer's read access |
| Portal layout edits lost after redeploy | Copy positions from **Export > Download** into the script first |

---

## Appendix A — Dashboard Generator Script

Python 3, no dependencies. Change `parts` to add, remove or move tiles; the header links are generic portal URLs.

```python
"""Generate an "Ops - Live" Azure portal dashboard: Resource Graph tiles for a second-screen ops view.

Usage: python3 Build-OpsDashboard.py [--location uksouth] [--out ops-live.json] [--queries]
Deploy the JSON with api-version 2015-08-01-preview (see README.md). Newer API versions
expect lenses/parts as arrays and reject this object-keyed format.
"""
import argparse, json, os

TITLE = "Ops - Live"
ARG = "Extension/HubsExtension/PartType/"
SH = "desktopvirtualizationresources | where type =~ 'microsoft.desktopvirtualization/hostpools/sessionhosts' | extend Pool=tostring(split(name,'/')[0]), Host=tostring(split(name,'/')[1]), Status=tostring(properties.status), Sessions=toint(properties.sessions), Heartbeat=todatetime(properties.lastHeartBeat)"
VM = "resources | where type =~ 'microsoft.compute/virtualmachines' | extend Power=tostring(properties.extended.instanceView.powerState.code)"
ARC = "resources | where type =~ 'microsoft.hybridcompute/machines'"
ALERTS = "alertsmanagementresources | where type =~ 'microsoft.alertsmanagement/alerts' | extend e=properties.essentials | where tostring(e.monitorCondition) == 'Fired' and todatetime(e.startDateTime) > ago(24h)"

HEADER = """<div style='line-height:44px'>
<span style='font-size:20px;font-weight:600'>Ops &mdash; Live</span>
<span style='opacity:.7'>&nbsp;&nbsp;All subscriptions &middot; Resource Graph &middot; set Auto refresh in the toolbar</span>
<span style='float:right'>
<a href='https://portal.azure.com/#view/Microsoft_Azure_WVD/WvdManagerMenuBlade/~/hostpools' target='_blank'>AVD</a> &nbsp;|&nbsp;
<a href='https://portal.azure.com/#view/Microsoft_Azure_Automation/AutomationMenuBlade/~/overview' target='_blank'>Automation</a> &nbsp;|&nbsp;
<a href='https://portal.azure.com/#view/Microsoft_Azure_Automanage/UpdateCenterMenuBlade/~/overview' target='_blank'>Update Manager</a> &nbsp;|&nbsp;
<a href='https://portal.azure.com/#view/Microsoft_Azure_HybridCompute/AzureArcCenterBlade/~/servers' target='_blank'>Arc</a> &nbsp;|&nbsp;
<a href='https://portal.azure.com/#view/Microsoft_Azure_CostManagement/Menu/~/costanalysis' target='_blank'>Cost</a>
</span></div>"""


def pos(x, y, w, h):
    return {"x": x, "y": y, "colSpan": w, "rowSpan": h}


def md(x, y, w, h, content):
    return {"position": pos(x, y, w, h), "metadata": {"inputs": [], "type": ARG + "MarkdownPart",
            "settings": {"content": {"settings": {"content": content, "title": "", "subtitle": ""}}}}}


def arg(kind, x, y, w, h, title, query, chart=None):
    inputs = [{"name": "partTitle", "value": title}, {"name": "query", "value": query}]
    if chart is not None:
        inputs.append({"name": "chartType", "value": chart})
    return {"position": pos(x, y, w, h), "metadata": {"inputs": inputs, "type": ARG + kind}}


single = lambda x, y, title, q: arg("ArgQuerySingleValueTile", x, y, 4, 3, title, q)
grid = lambda x, y, w, h, title, q: arg("ArgQueryGridTile", x, y, w, h, title, q)
bar = lambda x, y, w, h, title, q: arg("ArgQueryChartTile", x, y, w, h, title, q, chart=1)

parts = [
    md(0, 0, 20, 1, HEADER),
    # KPI strip
    single(0, 1, "VMs running", VM + " | where Power == 'PowerState/running' | summarize ['Running']=count()"),
    single(4, 1, "AVD hosts available", SH + " | where Status == 'Available' | summarize ['Available']=count()"),
    single(8, 1, "AVD hosts unavailable", SH + " | where Status !in ('Available','Shutdown') | summarize ['Unavailable']=count()"),
    single(12, 1, "Arc servers not connected", ARC + " | where tostring(properties.status) != 'Connected' | summarize ['Not connected']=count()"),
    single(16, 1, "Alerts fired (24h)", ALERTS + " | summarize ['Fired']=count()"),
    # AVD
    bar(0, 4, 6, 4, "AVD session hosts by status", SH + " | summarize Hosts=count() by Status | order by Hosts desc"),
    grid(6, 4, 14, 4, "AVD session hosts (powered on)", SH + " | where Status != 'Shutdown' | project Pool, Host, Status, Sessions, Heartbeat | order by Status asc, Pool asc"),
    # Compute / hybrid / alerts
    grid(0, 8, 7, 4, "Running VMs", VM + " | where Power == 'PowerState/running' | project VM=name, Size=tostring(properties.hardwareProfile.vmSize), resourceGroup | order by VM asc"),
    grid(7, 8, 6, 4, "Arc servers", ARC + " | project Server=name, Status=tostring(properties.status), Changed=todatetime(properties.lastStatusChange) | order by Status desc"),
    grid(13, 8, 7, 4, "Alerts fired (24h)", ALERTS + " | project Alert=name, Sev=tostring(e.severity), Target=tostring(e.targetResourceName), Started=todatetime(e.startDateTime) | order by Started desc"),
    # Patching / health
    grid(0, 12, 7, 4, "Patch runs (30 days)", "patchinstallationresources | where type endswith '/patchinstallationresults' | extend Machine=tostring(split(id,'/')[8]), Finished=todatetime(properties.lastModifiedDateTime) | where Finished > ago(30d) | project Machine, Result=tostring(properties.status), Installed=toint(properties.installedPatchCount), Failed=toint(properties.failedPatchCount), Finished | order by Finished desc"),
    grid(7, 12, 7, 4, "Pending patches by machine", "patchassessmentresources | where type endswith '/patchassessmentresults' | extend Machine=tostring(split(id,'/')[8]), c=properties.availablePatchCountByClassification | project Machine, Critical=toint(c.critical), Security=toint(c.security), Other=toint(c.other), Assessed=todatetime(properties.lastModifiedDateTime) | order by Critical desc, Security desc"),
    grid(14, 12, 6, 4, "Azure service health (active)", "servicehealthresources | where type =~ 'microsoft.resourcehealth/events' | where tostring(properties.status) =~ 'Active' | project Type=tostring(properties.eventType), Issue=tostring(properties['title']), Since=todatetime(properties.impactStartTime)"),
]

here = os.path.dirname(os.path.abspath(__file__))
ap = argparse.ArgumentParser()
ap.add_argument("--location", default="uksouth")
ap.add_argument("--out", default=os.path.join(here, "ops-live.json"))
ap.add_argument("--queries", action="store_true", help="print each tile query, one per line, for testing")
args = ap.parse_args()

dash = {
    "location": args.location,
    "tags": {"hidden-title": TITLE},
    "properties": {
        "lenses": {"0": {"order": 0, "parts": {str(i): p for i, p in enumerate(parts)}}},
        "metadata": {"model": {"timeRange": {"value": {"relative": {"duration": 24, "timeUnit": 1}},
                                             "type": "MsPortalFx.Composition.Configuration.ValueTypes.TimeRange"}}},
    },
}
with open(args.out, "w") as f:
    json.dump(dash, f, indent=2)
    f.write("\n")

if args.queries:
    for p in parts:
        for i in p["metadata"]["inputs"]:
            if i["name"] == "query":
                print(i["value"])
```
