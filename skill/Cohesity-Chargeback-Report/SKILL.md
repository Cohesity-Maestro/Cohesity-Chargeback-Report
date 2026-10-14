---
name: "Cohesity-Chargeback-Report"
description: "Build the \"Cohesity Chargeback Report\" page from live Helios data via the helios-mcp connector: monthly storage charge for every protection group across all clusters, with a price per GiB and currency, a choice of what to bill on (storage consumed, storage including resiliency, or front-end data), cluster filter, group search, cluster overhead and empty-group toggles, share of charge by cluster, growth, and Copy as CSV. Publishes it as a live claude.ai Artifact and refreshes it every 4 hours. Use when the user asks for backup chargeback or showback, cost per protection group, storage cost by team or cluster, who consumes the most backup storage, or to refresh/schedule that page."
---

# Cohesity Chargeback Report

Builds a single-page chargeback from the Helios **Storage Consumption by Protection Groups** report (read through the **helios-mcp** connector), publishes it as a persistent **Artifact** (private page with a stable URL), and refreshes that same artifact **every 4 hours** via a scheduled task. The full page template is embedded at the end of this skill. Never invent numbers. If a tool fails or returns nothing, say so and show "n/a". Never write a loose static HTML file as the deliverable. Everything is read-only.

## What the page contains

Eyebrow "COHESITY HELIOS · STORAGE CONSUMPTION BY PROTECTION GROUPS · DATA PULLED <date>", title "Cohesity Chargeback Report" and a one-line summary that names the number of clusters. Everything below is calculated **in the browser** from the data embedded at generation time, so changing a price or a toggle is instant and needs no Helios call:

- **Controls**: price per GiB per month with a currency (`$ € £ ₹ ¥`), **Bill on** (Storage consumed, Incl. resiliency, Front-end data), Cluster, Find group (name, source, type or policy), Show cluster overhead rows, Show empty groups, Copy as CSV (and Download CSV where the page's `downloads` capability exists). The viewer's choices are remembered in localStorage.
- **KPI strip**: Monthly charge (with the yearly figure), Storage consumed or Front-end data (with front-end size and data reduction), Groups billed (with cluster count), Largest consumer (cluster, share of charge, charge per month).
- **Share of charge by cluster**: one stacked bar and a legend with each cluster's share and charge.
- **Table grouped by cluster**: a header row per cluster (group count, totals, share, charge) and one row per protection group: name with its source names underneath, type chip, policy, front-end data, storage consumed, data reduction, change, share of the total with a mini bar, and charge per month.
- Footnote: how the numbers are defined and what is excluded.

The template already implements all of this. Do not redesign it; fill the placeholders.

## Billing definitions (state them in the footnote, the template does)

| Basis | Helios field | Meaning |
|---|---|---|
| Storage consumed (default) | `scRetainedBytes` | Physical storage used by the group's retained data, after dedupe and compression |
| Incl. resiliency | `scResiliencyBytes` | The same, plus the extra copies the cluster keeps for resiliency (erasure coding or replication factor) |
| Front-end data | `dataIngestedRetainedBytes` | Logical size of the data protected and retained, before data reduction |

Sizes use binary units (GiB = 2^30 bytes). **Change** is the growth Helios reports for the same basis over the report's own window (`scRetainedBytesGrowth`, `scResiliencyBytesGrowth`, `dataIngestedRetainedBytesGrowth`); the window is Helios's default, not chosen by this skill. **Reduction** is Helios's `dataReduction` ratio. The price is whatever the viewer types: the skill never assumes a price list, converts currencies or produces an invoice. Present it as showback/chargeback input, not billing of record.

## Data steps

1. `dataprotect-get-clusters`: registered cluster count, names and `isConnectedToHelios`. The report covers the clusters Helios has data for; list any registered cluster missing from the report in the footnote.
2. `reporting-get-report-preview` with `id: "storage-consumption-group"`, `componentIds: ["400","401"]`, `dataComponentLimit: 5000`. Component `400` is the per-group table (one row per protection group and target); `401` is a glance bar whose `countGroupId` and `sumScResiliencyBytes` let you check the rows. The response is large (about 170K characters for 180 groups) and is saved to a file: process it with `python3 -I`, do not read it into the conversation. If the number of `400` rows equals `dataComponentLimit`, raise the limit and call again. Verify that the row count equals `401.countGroupId` and that the sum of `scResiliencyBytes` over the rows equals `401.sumScResiliencyBytes`; if not, say so in the final message.
3. Build one `rows` entry per `400` row, using exactly these keys (bytes are integers, ratios are numbers):

| Key | Source field | Notes |
|---|---|---|
| `c` | `system` | Cluster name |
| `g` | `groupName` | Empty for cluster overhead rows. Strip a leading `_DELETED_` and set `del: 1` |
| `src` | `sourceNames` | Array, keep at most 4 names; hostnames are fine, drop credentials and URLs with secrets |
| `env` | `environment` | Raw Helios code such as `kVMware`; the page maps it to a label |
| `pol` | `policyName` | |
| `fe` | `dataIngestedRetainedBytes` | |
| `sc` | `scRetainedBytes` | |
| `scr` | `scResiliencyBytes` | |
| `red` | `dataReduction` | |
| `gFe`, `gSc`, `gScr` | the three `...BytesGrowth` fields | Change per basis |
| `oh` | derived | `1` when `groupName` is empty: storage Helios could not attribute to a protection group (cluster overhead). Hidden by the page unless the viewer ticks "Show cluster overhead rows" |
| `del` | derived | `1` for `_DELETED_` groups: the group was deleted but its snapshots still hold storage; the page shows a "deleted" chip |

   Rows from a replication or archive target (`targetRole`/`targetName` other than `Local`) are separate rows with the same group name; keep them, and add `tgt` with the target name when it is not `Local` (the page shows it next to the policy).
4. `{{PULLED_DATE}}` is the report's `generatedTimestampUsecs` formatted in UTC with the time, as `13 OCT 2026, 23:57 UTC`; `{{GENERATED_UTC}}` is the current time.

If the report is unavailable (for example the user's role cannot run reports), say so and stop: there is no other Helios MCP tool that returns storage consumption per protection group, and per-run `bytesWritten` is not a substitute.

## Filling the template

| Placeholder | Value |
|---|---|
| `{{PULLED_DATE}}` | date and time in UTC, e.g. `13 OCT 2026, 23:57 UTC` |
| `{{GENERATED_UTC}}` | e.g. `2026-10-13 22:40 UTC` |
| `{{DATA_JSON}}` | `{"registered":12,"rows":[...]}` as valid JSON with every `</` written as `<\/` |
| `{{SCOPE_NOTE}}` | footnote text, HTML-escaped: clusters registered but absent from the report, row-count check result, any cap |

## Manual refresh

The template has a **Refresh from Helios** button. A published page cannot call the Helios connector itself, so the second button copies the command `Run the Cohesity-Chargeback-Report skill to refresh this report from live Helios data`; paste it into Claude and it republishes the same artifact. Keep the button in every rebuild.

## When Helios is unavailable

Make `dataprotect-get-clusters` the first Helios call. If it fails ("Server helios-mcp unavailable", an expired token, or a timeout twice in a row), **stop collecting data and do not clear, blank or overwrite anything the page already holds**. Instead show an error on the existing page:

1. Find the live artifact (its URL is in the scheduled task prompt, or use `Artifact` `action: "list"` and match the title "Cohesity Chargeback Report" case-insensitively) and read it with `action: "read"`, `path: "index.html"`, `out_dir` set to the scratch directory. If no artifact exists yet, publish nothing and tell the user.
2. Build an HTML **fragment** from the saved page: everything from the first `<title>` up to (not including) `</body>`. Leave the data JSON, scripts and styles exactly as they are.
3. Add or replace this banner directly after `</header>` (in the Chargeback Report, directly after the `<h1>`; replace the `<!--HELIOS_ERR-->` marker, or an existing `<div class="errbar"` block if one is already there; add the `.errbar` rule to the `<style>` block if the page lacks it):
   `<div class="errbar" role="alert">Unable to connect to Helios. Showing the last successful data. Last attempt <UTC time>. Reconnect the helios-mcp connector, then refresh.</div>`
4. Publish the fragment with `url` set to the artifact, `capabilities omitted` (the stored declaration is kept), so the page keeps showing its last data plus the error message.
5. Reply in one line: "Unable to connect to Helios; kept the last data and added an error banner."

A later successful run rebuilds the page from the template, which carries an empty `<!--HELIOS_ERR-->` marker, so the banner disappears by itself. Never invent or reuse old numbers as new ones: the banner is the only change on a failed run.

## Output: publish as a live Artifact

1. Write the filled template to a scratch file as an HTML **fragment** (no doctype/html/head/body) in the session's working/outputs directory (for example `<outputs>/_build/chargeback.html`) or any scratch folder the Write tool can reach, never inside a git repository; keep `<title>Cohesity Chargeback Report</title>` stable. If the `artifact-design` skill is available, load it first.
2. Check whether it already exists: `Artifact` with `action: "list"`, title "Cohesity Chargeback Report".
   - **Exists** (any refresh): `action: "read"` it, then publish the new file with `url` set to it. Never create a second artifact, do not pass `icon`, omit `capabilities` so the stored declaration is kept.
   - **Does not exist**: publish with `icon: "chart"`, `capabilities: {downloads: true}` and a one-sentence `description`.
   - **Stored URL not found** (the artifact was deleted or is not visible to this account): do not stop. Publish the page as a new artifact (first-publish rules above) and tell the user in the reply: "I couldn't find the previous artifact, so I republished a new one", with the new link, and that any scheduled task prompt still pointing at the old URL needs the new one.
3. The Artifact is the deliverable. Do not also save a static HTML file.
4. Reply in 2 to 3 sentences: data-pulled date and time, the monthly charge at the default price of 1 per GiB, the largest consumer, and anything that did not tie. On a scheduled refresh, one line.

## Refresh every 4 hours (default)

After the first publish, call `mcp__scheduled-tasks__list_scheduled_tasks` and look for a task whose title contains "chargeback". If missing, create `cohesity-chargeback-report-refresh` with title "Cohesity Chargeback Report (every 4 hours)", `cronExpression: "0 */4 * * *"` (local time), `notifyOnCompletion: false`, and this self-contained prompt: "Run the Cohesity-Chargeback-Report skill end to end with live helios-mcp data (read-only) and republish the existing 'Cohesity Chargeback Report' artifact: find it with the Artifact list action by title (case-insensitive) or use its URL if one is given here (add it when the task is created: ARTIFACT_URL), read it, then publish the refreshed page with its url. Do not pass an icon or capabilities. Never create a second artifact. Reply in one line with the data-pulled date and time and the monthly charge at 1 per GiB. If helios-mcp is unavailable, follow the skill's \"When Helios is unavailable\" section (keep the last data and add the error banner). If the skill is not installed in Claude, say so and stop; install it from github.com/Cohesity-Maestro/Cohesity-Chargeback-Report." The refresh is cheap (two Helios calls) because the report does the aggregation. If a task exists with another cadence, fix it with `update_scheduled_task`. Tell the user scheduled tasks run only while the Claude app is open, a missed run executes on next launch, and the task runs in Auto-approve mode unless they edit it. The viewer's price, currency and toggles survive a refresh because they live in the viewer's browser. Keep the `refreshes every 4 hours` text in the template in sync with the cron.

## Rules
- Read-only against Helios: only get/list/preview tools.
- Do not assume a price, convert currencies or call the output an invoice.
- Cluster overhead is shown only on request and is never silently added to a team's charge.
- The artifact stays private; tell the user to share it from the page's Share menu.

## Page template

```html
<title>Cohesity Chargeback Report</title>
<style>
/* Layout: eyebrow, title, controls panel, KPI strip, share-by-cluster bar, grouped table. A calculator over report data: the numbers are the page. */
:root{--bg:#f2f5f4;--card:#ffffff;--border:#d9e2df;--text:#1d2927;--muted:#5a6a66;--rule:#e9eeec;--track:#e8eeec;--hover:#f2f7f5;--head:#e8f1ee;
--accent:#c26a12;--ctl:#ffffff;--sel:#1f7a6d;--sel-text:#ffffff;
--s1:#2a9d8f;--s2:#6a7fd0;--s3:#d98a2b;--s4:#b05fb0;--s5:#b8962a;--s6:#4f86c6}
@media (prefers-color-scheme: dark){:root:not([data-theme="light"]){--bg:#0f1715;--card:#151f1c;--border:#26352f;--text:#e6eeec;--muted:#93a8a2;--rule:#1e2b27;--track:#22302b;--hover:#1a2723;--head:#1a2c27;
--accent:#e9a24e;--ctl:#0f1715;--sel:#5fd0bf;--sel-text:#0b1a17;
--s1:#4fd0c0;--s2:#8fa2e8;--s3:#e9a24e;--s4:#d58fd5;--s5:#d9b94a;--s6:#78abe8;color-scheme:dark}}
:root[data-theme="dark"]{--bg:#0f1715;--card:#151f1c;--border:#26352f;--text:#e6eeec;--muted:#93a8a2;--rule:#1e2b27;--track:#22302b;--hover:#1a2723;--head:#1a2c27;
--accent:#e9a24e;--ctl:#0f1715;--sel:#5fd0bf;--sel-text:#0b1a17;
--s1:#4fd0c0;--s2:#8fa2e8;--s3:#e9a24e;--s4:#d58fd5;--s5:#d9b94a;--s6:#78abe8;color-scheme:dark}
body{background:var(--bg);color:var(--text);font-family:system-ui,-apple-system,"Segoe UI",Roboto,Arial,sans-serif;padding-inline:16px}
*{box-sizing:border-box}
.wrap{max-width:1240px;margin:0 auto;padding-block:26px 44px}
.mono{font-family:ui-monospace,"SF Mono",Menlo,Consolas,monospace}
.eyebrow{font-size:11px;letter-spacing:.14em;text-transform:uppercase;color:var(--muted)}
h1{margin:6px 0 6px;font-size:32px;letter-spacing:-.01em;text-wrap:balance}
.lede{margin:0 0 18px;color:var(--muted);font-size:15px;line-height:1.5;max-width:70ch}
.ctl{display:flex;flex-wrap:wrap;gap:16px 22px;align-items:flex-end;background:var(--card);border:1px solid var(--border);border-radius:12px;padding:16px 18px;margin-bottom:16px}
.fld{display:flex;flex-direction:column;gap:6px;min-width:0}
.fld>label,.fld>.fl{font-size:11px;letter-spacing:.1em;text-transform:uppercase;color:var(--muted)}
.row{display:flex;gap:0}
select,input[type=number],input[type=search]{background:var(--ctl);color:var(--text);border:1px solid var(--border);border-radius:8px;padding:9px 10px;font:inherit;font-size:14px;min-width:0}
input[type=number]{width:120px;border-radius:0 8px 8px 0;border-left:0}
#cur{border-radius:8px 0 0 8px;width:64px}
#cl{min-width:190px}
input[type=search]{width:220px}
.seg{display:flex;border:1px solid var(--border);border-radius:8px;overflow:hidden}
.seg button{background:var(--ctl);color:var(--text);border:0;border-right:1px solid var(--border);padding:9px 13px;font:inherit;font-size:13px;cursor:pointer}
.seg button:last-child{border-right:0}
.seg button[aria-pressed="true"]{background:var(--sel);color:var(--sel-text);font-weight:600}
.chk{display:flex;align-items:center;gap:8px;font-size:14px;padding-block:9px}
.chk input{width:16px;height:16px;accent-color:var(--sel)}
.btn{background:var(--card);color:var(--text);border:1px solid var(--border);border-radius:8px;padding:9px 14px;font:inherit;font-size:13px;font-weight:600;cursor:pointer}
.btn:hover{background:var(--hover)}
.push{margin-left:auto;display:flex;gap:8px}
select:focus-visible,input:focus-visible,.seg button:focus-visible,.btn:focus-visible{outline:2px solid var(--sel);outline-offset:2px}
.kpis{display:grid;grid-template-columns:repeat(4,minmax(0,1fr));background:var(--card);border:1px solid var(--border);border-radius:12px;margin-bottom:22px}
.kpi{padding:16px 18px;border-right:1px solid var(--border);min-width:0}
.kpi:last-child{border-right:0}
.kpi .k{font-size:11px;letter-spacing:.1em;text-transform:uppercase;color:var(--muted)}
.kpi .v{font-size:30px;font-weight:700;margin:6px 0 4px;font-variant-numeric:tabular-nums;overflow-wrap:anywhere}
.kpi .v.money{color:var(--accent)}
.kpi .v.name{font-size:20px;line-height:1.25}
.kpi .s{font-size:12px;color:var(--muted);line-height:1.5}
@media(max-width:900px){.kpis{grid-template-columns:repeat(2,minmax(0,1fr))}.kpi:nth-child(2){border-right:0}.kpi:nth-child(-n+2){border-bottom:1px solid var(--border)}}
@media(max-width:520px){.kpis{grid-template-columns:minmax(0,1fr)}.kpi{border-right:0;border-bottom:1px solid var(--border)}.kpi:last-child{border-bottom:0}.push{margin-left:0}}
h2{font-size:18px;margin:0 0 10px}
.share{display:flex;height:14px;border-radius:7px;overflow:hidden;background:var(--track);margin-bottom:10px}
.share i{display:block;height:100%}
.legend{display:flex;flex-wrap:wrap;gap:6px 20px;font-size:12px;color:var(--muted);margin-bottom:22px}
.legend span{display:inline-flex;align-items:center;gap:7px}
.legend b{display:inline-block;width:10px;height:10px;border-radius:2px}
.legend em{font-style:normal;color:var(--text)}
.tw{overflow-x:auto;border:1px solid var(--border);border-radius:12px;background:var(--card)}
.tw.sc{max-height:620px;overflow:auto}
.tw.sc thead th{position:sticky;top:0;background:var(--card);z-index:2}
tr.ch{cursor:pointer}
tr.ch:focus-visible{outline:2px solid var(--sel);outline-offset:-2px}
table{width:100%;border-collapse:collapse;font-size:14px;min-width:1040px}
th{text-align:left;font-size:11px;letter-spacing:.1em;text-transform:uppercase;color:var(--muted);font-weight:500;padding:12px 12px;border-bottom:1px solid var(--border);white-space:nowrap}
th.r,td.r{text-align:right}
td{padding:11px 12px;border-bottom:1px solid var(--rule);vertical-align:middle}
td.n{font-variant-numeric:tabular-nums;white-space:nowrap}
tr.ch td{background:var(--head);font-weight:700}
tr.ch td .sm{font-weight:400;color:var(--muted);font-size:13px}
tr.gr:hover td{background:var(--hover)}
.gn{font-weight:600;overflow-wrap:anywhere}
.gs{font-size:12px;color:var(--muted);overflow-wrap:anywhere;max-width:260px}
.chip{display:inline-block;border:1px solid var(--border);border-radius:6px;padding:3px 8px;font-size:11px;white-space:nowrap}
.chip.del{border-color:var(--accent);color:var(--accent)}
.mini{display:inline-block;vertical-align:middle;width:78px;height:6px;border-radius:3px;background:var(--track);margin-left:8px;overflow:hidden}
.mini i{display:block;height:100%;background:var(--s1)}
.up{color:var(--accent)}.dn{color:var(--sel)}
.dim{color:var(--muted)}
.empty{padding:22px;color:var(--muted);text-align:center}
.foot{font-size:12px;color:var(--muted);line-height:1.6;margin-top:14px;max-width:110ch}
.errbar{background:var(--card,#fff);color:var(--red,#c0392b);border:1px solid var(--red,#c0392b);border-left:4px solid var(--red,#c0392b);border-radius:8px;padding:10px 14px;margin:0 0 16px;font-size:13px;line-height:1.5}
.rfr{display:flex;justify-content:flex-end;align-items:center;gap:8px;flex-wrap:wrap;margin:0 0 12px;font-size:12px;color:var(--muted,#5b6a67)}.rfr button{background:var(--card,#fff);color:var(--text,#1f2a28);border:1px solid var(--border,#dbe3e0);border-radius:6px;padding:6px 12px;font-size:12px;cursor:pointer}.rfr button:focus-visible{outline:2px solid var(--green,#2e7d32);outline-offset:2px}.rfr input{font:12px "Courier New",monospace;background:var(--card,#fff);color:var(--text,#1f2a28);border:1px solid var(--border,#dbe3e0);border-radius:6px;padding:5px 8px;width:300px;max-width:100%}
</style>
<div class="wrap">
<div class="eyebrow mono">Cohesity Helios · Storage consumption by protection groups · Data pulled {{PULLED_DATE}}</div>
<h1>Cohesity Chargeback Report</h1>
<!--HELIOS_ERR-->
<div class="rfr"><span id="rfn">Auto-refreshes every 4 hours.</span><button type="button" id="rf" title="Copy the command that pulls live data from Helios">Refresh from Helios</button></div>
<script>
(function(){
var CMD='Run the Cohesity-Chargeback-Report skill to refresh this report from live Helios data';
var n=document.getElementById('rfn');
document.getElementById('rf').addEventListener('click',function(){
 function done(ok){if(ok){n.textContent='Copied. Paste it into Claude to pull live data; reopen or refresh this tab when it finishes.'}else{n.textContent='';var i=document.createElement('input');i.readOnly=true;i.value=CMD;i.setAttribute('aria-label','Refresh command');n.appendChild(document.createTextNode('Copy this into Claude; reopen or refresh this tab when it finishes: '));n.appendChild(i);i.focus();i.select()}}
 try{navigator.clipboard.writeText(CMD).then(function(){done(true)},function(){done(false)})}catch(e){done(false)}
});
})();
</script>
<p class="lede" id="lede"></p>

<div class="ctl">
 <div class="fld"><label for="price">Price per GiB / month</label>
  <div class="row"><select id="cur" aria-label="Currency"><option value="$">$</option><option value="€">€</option><option value="£">£</option><option value="₹">₹</option><option value="¥">¥</option></select><input type="number" id="price" min="0" step="0.01" value="1"></div></div>
 <div class="fld"><span class="fl">Bill on</span>
  <div class="seg" role="group" aria-label="Bill on"><button type="button" data-b="sc" aria-pressed="true">Storage consumed</button><button type="button" data-b="scr" aria-pressed="false">Incl. resiliency</button><button type="button" data-b="fe" aria-pressed="false">Front-end data</button></div></div>
 <div class="fld"><label for="cl">Cluster</label><select id="cl"></select></div>
 <div class="fld"><label for="q">Find group</label><input type="search" id="q" placeholder="Name, source, type…"></div>
 <div class="fld"><label for="ty">Type</label><select id="ty"></select></div>
 <div class="fld"><label for="po">Policy</label><select id="po"></select></div>
 <div class="fld"><label for="so">Sort groups by</label><select id="so"><option value="charge">Charge (high to low)</option><option value="fe">Front-end data</option><option value="chg">Change (growth)</option><option value="red">Reduction (best first)</option><option value="name">Name (A to Z)</option></select></div>
 <div class="fld"><label for="tn">Per cluster</label><select id="tn"><option value="all">All groups</option><option value="5">Top 5</option><option value="10">Top 10</option><option value="25">Top 25</option></select></div>
 <label class="chk"><input type="checkbox" id="oh"> Show cluster overhead rows</label>
 <label class="chk"><input type="checkbox" id="em"> Show empty groups</label>
 <div class="push"><button type="button" class="btn" id="ca">Collapse all</button><button type="button" class="btn" id="copy">Copy as CSV</button><button type="button" class="btn" id="dl" hidden>Download CSV</button></div>
</div>

<div class="kpis">
 <div class="kpi"><div class="k mono">Monthly charge</div><div class="v money" id="k1">-</div><div class="s" id="k1s"></div></div>
 <div class="kpi"><div class="k mono" id="k2l">Storage consumed</div><div class="v" id="k2">-</div><div class="s" id="k2s"></div></div>
 <div class="kpi"><div class="k mono">Groups billed</div><div class="v" id="k3">-</div><div class="s" id="k3s"></div></div>
 <div class="kpi"><div class="k mono">Largest consumer</div><div class="v name" id="k4">-</div><div class="s" id="k4s"></div></div>
</div>

<h2>Share Of Charge By Cluster</h2>
<div class="share" id="share" role="img" aria-label="Share of charge by cluster"></div>
<div class="legend" id="legend"></div>

<div class="tw sc"><table><thead><tr><th>Protection group</th><th>Type</th><th>Policy</th><th class="r">Front-end data</th><th class="r" id="th2">Storage consumed</th><th class="r">Reduction</th><th class="r">Change</th><th class="r">Share</th><th class="r" id="thc">Charge / mo</th></tr></thead><tbody id="tb"></tbody></table></div>
<div class="foot" id="foot"></div>
<div class="foot">Source: Helios MCP, read-only. Generated {{GENERATED_UTC}}, refreshes every 4 hours. Prices, currency and toggles are calculated in this page and remembered in your browser; they do not change Helios. Showback input, not an invoice. Sizes use binary units (1 GiB = 2<sup>30</sup> bytes). Change is the growth Helios reports for the chosen basis over the report's own window. {{SCOPE_NOTE}}</div>
</div>

<script id="data" type="application/json">
{{DATA_JSON}}
</script>
<script>
(function(){
var D=JSON.parse(document.getElementById('data').textContent);
var ENV={kVMware:"VMware",kView:"View / SmartFiles",kPhysicalFiles:"Physical files",kPhysical:"Physical",kSQL:"SQL Server",kOracle:"Oracle",kAD:"Active Directory",kGenericNas:"NAS",kNetapp:"NetApp",kIsilon:"Isilon",kPure:"Pure",kAcropolis:"Nutanix AHV",kHyperV:"Hyper-V",kKubernetes:"Kubernetes",kUDA:"UDA",kPostgres:"PostgreSQL",kMongoDBPhysical:"MongoDB",kPuppeteer:"Puppeteer",kO365Exchange:"M365 Exchange",kO365OneDrive:"M365 OneDrive",kO365Sharepoint:"M365 SharePoint",kO365Teams:"M365 Teams",kAWSNative:"AWS",kAwsDynamoDB:"DynamoDB",kAWSRDSMSSQL:"AWS RDS SQL",kAWSRDSPostgresDB:"AWS RDS Postgres",kS3Compatible:"S3",kSAPHANA:"SAP HANA",kDB2:"Db2"};
var BASIS={sc:{vol:'sc',g:'gSc',name:'Storage consumed'},scr:{vol:'scr',g:'gScr',name:'Incl. resiliency'},fe:{vol:'fe',g:'gFe',name:'Front-end data'}};
var st={cur:'$',price:1,b:'sc',cl:'',q:'',ty:'',po:'',so:'charge',tn:'all',oh:false,em:false};
var collapsed={};
try{var s=JSON.parse(localStorage.getItem('cchr-state')||'null');if(s){for(var k in st)if(k in s&&k!=='cl'&&k!=='q'&&k!=='ty'&&k!=='po')st[k]=s[k]}}catch(e){}
var $=function(i){return document.getElementById(i)};
var COL=['var(--s1)','var(--s2)','var(--s3)','var(--s4)','var(--s5)','var(--s6)'];
var G=1073741824;
function el(tag,cls,txt){var e=document.createElement(tag);if(cls)e.className=cls;if(txt!==undefined)e.textContent=txt;return e}
function num(v){return Number(v.toFixed(2)).toString()}
function bytes(b){if(!b)return'0 B';var u=['B','KiB','MiB','GiB','TiB','PiB'],i=0,v=Math.abs(b);while(v>=1024&&i<u.length-1){v/=1024;i++}return(b<0?'-':'')+num(v)+' '+u[i]}
function money(v){return st.cur+v.toLocaleString('en-US',{minimumFractionDigits:2,maximumFractionDigits:2})}
function pct(v){return(Math.round(v*10)/10).toFixed(1)+'%'}
function sources(r){var a=r.src||[];return a.length>3?a.slice(0,3).join(', ')+' +'+(a.length-3):a.join(', ')}
var clusters=[];D.rows.forEach(function(r){if(clusters.indexOf(r.c)<0)clusters.push(r.c)});
clusters.sort();
var cs=$('cl');var o=el('option','','All clusters');o.value='';cs.appendChild(o);
clusters.forEach(function(c){var x=el('option','',c);x.value=c;cs.appendChild(x)});
$('cur').value=st.cur;$('price').value=st.price;$('oh').checked=st.oh;$('em').checked=st.em;
var shown=[];
function save(){try{localStorage.setItem('cchr-state',JSON.stringify({cur:st.cur,price:st.price,b:st.b,oh:st.oh,em:st.em,so:st.so,tn:st.tn}))}catch(e){}}
function calc(){
 var B=BASIS[st.b],q=st.q.trim().toLowerCase(),price=st.price>0?st.price:0;
 var rows=D.rows.filter(function(r){
  if(st.cl&&r.c!==st.cl)return false;
  if(r.oh&&!st.oh)return false;
  if(st.ty&&r.env!==st.ty)return false;
  if(st.po&&r.pol!==st.po)return false;
  if(!st.em&&!(r[B.vol]>0))return false;
  if(q){var h=((r.g||'')+' '+sources(r)+' '+(ENV[r.env]||r.env||'')+' '+(r.pol||'')).toLowerCase();if(h.indexOf(q)<0)return false}
  return true});
 var tot=0;rows.forEach(function(r){r.$=r[B.vol]/G*price;tot+=r.$});
 var by={};rows.forEach(function(r){var c=by[r.c]=by[r.c]||{n:r.c,rows:[],fe:0,sc:0,v:0,chg:0,$:0};c.rows.push(r);c.fe+=r.fe;c.sc+=(st.b==='scr'?r.scr:r.sc);c.v+=r[B.vol];c.chg+=r[B.g]||0;c.$+=r.$});
 var list=Object.keys(by).map(function(k){return by[k]}).sort(function(a,b){return b.$-a.$||b.v-a.v});
 var cmp={charge:function(a,b){return b.$-a.$||b[B.vol]-a[B.vol]},fe:function(a,b){return b.fe-a.fe},chg:function(a,b){return(b[B.g]||0)-(a[B.g]||0)},red:function(a,b){return(b.red||0)-(a.red||0)},name:function(a,b){return(a.g||'').localeCompare(b.g||'')}}[st.so]||function(a,b){return b.$-a.$};
 list.forEach(function(c){c.rows.sort(cmp)});
 return{B:B,rows:rows,tot:tot,list:list,price:price}
}
function chg(v){if(!v)return el('span','dim','—');var s=el('span',v>0?'up':'dn',(v>0?'+':'−')+bytes(Math.abs(v)));return s}
function render(){
 var R=calc(),B=R.B;shown=R.rows;
 var ncl=R.list.length,groups=R.rows.filter(function(r){return !r.oh&&r[B.vol]>0});
 $('lede').textContent='Monthly storage charge for every protection group across '+(st.cl?'cluster '+st.cl:'your '+clusters.length+' cluster'+(clusters.length===1?'':'s'))+'. Set a price per GiB and choose what to bill on; the totals and the CSV update as you go.';
 var vol=0,fe=0,sc=0;R.rows.forEach(function(r){vol+=r[B.vol];fe+=r.fe;sc+=r.sc});
 $('k1').textContent=money(R.tot);$('k1s').textContent=money(R.tot*12)+' per year at '+st.cur+num(R.price)+'/GiB';
 $('k2l').textContent=B.name==='Incl. resiliency'?'Storage incl. resiliency':B.name;
 $('k2').textContent=(vol/G).toLocaleString('en-US',{minimumFractionDigits:2,maximumFractionDigits:2})+' GiB';
 $('k2s').textContent=st.b==='fe'?('stored as '+bytes(sc)+(sc?' · '+num(fe/sc)+'× reduction':'')):('from '+bytes(fe)+' front-end'+(sc?' · '+num(fe/sc)+'× reduction':''));
 $('k3').textContent=groups.length;$('k3s').textContent='across '+ncl+' cluster(s)';
 var top=groups.slice().sort(function(a,b){return b.$-a.$})[0];
 $('k4').textContent=top?top.g:'—';$('k4s').textContent=top?top.c+' · '+pct(R.tot?top.$/R.tot*100:0)+' · '+money(top.$)+'/mo':'';
 $('th2').textContent=st.b==='scr'?'Storage incl. resiliency':'Storage consumed';
 $('thc').textContent='Charge / mo';
 var sh=$('share'),lg=$('legend');sh.textContent='';lg.textContent='';
 R.list.forEach(function(c,i){var col=COL[clusters.indexOf(c.n)%COL.length];
  var w=R.tot?c.$/R.tot*100:0;var b=el('i');b.style.width=w+'%';b.style.background=col;b.title=c.n+' '+pct(w);sh.appendChild(b);
  var sp=el('span');var sw=el('b');sw.style.background=col;sp.appendChild(sw);sp.appendChild(el('em','',c.n));sp.appendChild(el('span','mono',pct(w)+' · '+money(c.$)));lg.appendChild(sp)});
 var tb=$('tb');tb.textContent='';
 R.list.forEach(function(c){
  var isC=!!collapsed[c.n],tnN=st.tn==='all'?1e9:parseInt(st.tn,10);
  var h=el('tr','ch');h.tabIndex=0;h.setAttribute('aria-expanded',isC?'false':'true');
  (function(n){function tg(){collapsed[n]=!collapsed[n];render()}h.addEventListener('click',tg);h.addEventListener('keydown',function(e){if(e.key==='Enter'||e.key===' '){e.preventDefault();tg()}})})(c.n);
  var a=el('td');a.appendChild(document.createTextNode((isC?'▸ ':'▾ ')+c.n+' '));a.appendChild(el('span','sm','· '+c.rows.length+' group'+(c.rows.length===1?'':'s')));h.appendChild(a);
  h.appendChild(el('td'));h.appendChild(el('td'));
  h.appendChild(el('td','n r',bytes(c.fe)));h.appendChild(el('td','n r',bytes(c.sc)));h.appendChild(el('td'));
  var cc=el('td','n r');if(c.chg)cc.appendChild(chg(c.chg));else cc.textContent='—';h.appendChild(cc);
  h.appendChild(el('td','n r',pct(R.tot?c.$/R.tot*100:0)));h.appendChild(el('td','n r up',money(c.$)));tb.appendChild(h);
  (isC?[]:c.rows.slice(0,tnN)).forEach(function(r){var t=el('tr','gr');
   var a=el('td');a.appendChild(el('div','gn',r.oh?'Cluster overhead (not attributed to a group)':r.g));var sub=r.oh?'':sources(r);if(sub)a.appendChild(el('div','gs',sub));t.appendChild(a);
   var ty=el('td');if(!r.oh)ty.appendChild(el('span','chip mono',ENV[r.env]||(r.env||'').replace(/^k/,'')));if(r.del){ty.appendChild(document.createTextNode(' '));ty.appendChild(el('span','chip mono del','deleted'))}t.appendChild(ty);
   t.appendChild(el('td','',(r.pol||'')+(r.tgt?' → '+r.tgt:'')));
   t.appendChild(el('td','n r',bytes(r.fe)));t.appendChild(el('td','n r',bytes(st.b==='scr'?r.scr:r.sc)));
   t.appendChild(el('td','n r'+(r.red?'':' dim'),r.red?num(r.red)+'×':'—'));
   var g=el('td','n r');g.appendChild(chg(r[B.g]||0));t.appendChild(g);
   var sh2=R.tot?r.$/R.tot*100:0;var s=el('td','n r');s.appendChild(document.createTextNode(pct(sh2)));var m=el('span','mini'),mi=el('i');mi.style.width=Math.min(sh2,100)+'%';mi.style.background=COL[clusters.indexOf(r.c)%COL.length];m.appendChild(mi);s.appendChild(m);t.appendChild(s);
   t.appendChild(el('td','n r',money(r.$)));tb.appendChild(t)});
  if(!isC&&c.rows.length>tnN){var mr=el('tr'),md=el('td','empty','+ '+(c.rows.length-tnN)+' more group(s) in '+c.n+' hidden by the Per cluster filter');md.colSpan=9;mr.appendChild(md);tb.appendChild(mr)}});
 var allC=R.list.length>0&&R.list.every(function(c){return collapsed[c.n]});$('ca').textContent=allC?'Expand all':'Collapse all';
 if(!R.list.length){var e=el('tr'),d=el('td','empty','No protection groups match these filters.');d.colSpan=9;e.appendChild(d);tb.appendChild(e)}
 var hid=D.rows.filter(function(r){return r.oh}).length,emp=D.rows.filter(function(r){return !r.oh&&!(r[B.vol]>0)}).length;
 $('foot').textContent=(st.oh?'':hid+' cluster overhead row'+(hid===1?'':'s')+' hidden. ')+(st.em?'':emp+' empty group'+(emp===1?'':'s')+' hidden. ')+'Deleted groups still hold storage through their snapshots and are billed until those expire.';
}
function csv(){
 var B=BASIS[st.b];function q(v){v=v==null?'':String(v);return /[",\n]/.test(v)?'"'+v.replace(/"/g,'""')+'"':v}
 var out=['cluster,group,type,policy,front_end_gib,storage_consumed_gib,reduction,change_gib,share_pct,charge_per_month,currency,bill_on'];
 var R=calc();R.list.forEach(function(c){c.rows.forEach(function(r){
  out.push([c.n,r.oh?'(cluster overhead)':r.g,ENV[r.env]||r.env||'',r.pol||'',(r.fe/G).toFixed(3),((st.b==='scr'?r.scr:r.sc)/G).toFixed(3),r.red?r.red.toFixed(2):'',((r[B.g]||0)/G).toFixed(3),(R.tot?r.$/R.tot*100:0).toFixed(2),r.$.toFixed(2),st.cur,B.name].map(q).join(','))})});
 return out.join('\n')}
function flash(b,t){var o=b.textContent;b.textContent=t;setTimeout(function(){b.textContent=o},1400)}
$('copy').addEventListener('click',function(){var b=this,t=csv();
 function fb(){var ta=document.createElement('textarea');ta.value=t;ta.style.position='fixed';ta.style.opacity='0';document.body.appendChild(ta);ta.select();var ok=false;try{ok=document.execCommand('copy')}catch(e){}document.body.removeChild(ta);flash(b,ok?'Copied':'Copy blocked')}
 try{navigator.clipboard.writeText(t).then(function(){flash(b,'Copied')},fb)}catch(e){fb()}});
(function(){var ty=[],po=[];D.rows.forEach(function(r){if(!r.oh&&r.env&&ty.indexOf(r.env)<0)ty.push(r.env);if(!r.oh&&r.pol&&po.indexOf(r.pol)<0)po.push(r.pol)});
 ty.sort(function(a,b){return(ENV[a]||a).localeCompare(ENV[b]||b)});po.sort();
 var t=$('ty'),p=$('po'),o1=el('option','','All types'),o2=el('option','','All policies');o1.value='';o2.value='';t.appendChild(o1);p.appendChild(o2);
 ty.forEach(function(v){var o=el('option','',ENV[v]||v.replace(/^k/,''));o.value=v;t.appendChild(o)});po.forEach(function(v){var o=el('option','',v);o.value=v;p.appendChild(o)});
 $('so').value=st.so;$('tn').value=st.tn;
 t.addEventListener('change',function(){st.ty=this.value;render()});p.addEventListener('change',function(){st.po=this.value;render()});
 $('so').addEventListener('change',function(){st.so=this.value;save();render()});$('tn').addEventListener('change',function(){st.tn=this.value;save();render()});
 $('ca').addEventListener('click',function(){var names=clusters.slice();var allC=names.every(function(n){return collapsed[n]});names.forEach(function(n){collapsed[n]=!allC});render()})})();
$('cur').addEventListener('change',function(){st.cur=this.value;save();render()});
$('price').addEventListener('input',function(){st.price=parseFloat(this.value)||0;save();render()});
$('cl').addEventListener('change',function(){st.cl=this.value;render()});
$('q').addEventListener('input',function(){st.q=this.value;render()});
$('oh').addEventListener('change',function(){st.oh=this.checked;save();render()});
$('em').addEventListener('change',function(){st.em=this.checked;save();render()});
Array.prototype.forEach.call(document.querySelectorAll('.seg button'),function(b){
 b.setAttribute('aria-pressed',b.dataset.b===st.b?'true':'false');
 b.addEventListener('click',function(){st.b=b.dataset.b;Array.prototype.forEach.call(document.querySelectorAll('.seg button'),function(x){x.setAttribute('aria-pressed',x.dataset.b===st.b?'true':'false')});save();render()})});
render();
(async function(){try{var dl=await claude.use('downloads');if(!dl)return;var b=$('dl');b.hidden=false;
 b.addEventListener('click',async function(){try{await dl.save({filename:'cohesity-chargeback-report.csv',data:csv()})}catch(e){}})}catch(e){}})();
})();
</script>
```
