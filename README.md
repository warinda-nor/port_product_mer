# Port Fitting Overview — Tableau Dashboard Extension

Live version of the "Port Fitting Overview" dashboard. Instead of static numbers baked
into the file, this pulls data directly from a Tableau worksheet at runtime and
recomputes every chart client-side (grouping, YoY comparison, top-N rankings, etc.).

## Files in this folder

| File | Purpose |
|---|---|
| `index.html` | The extension itself — same look as the static dashboard, but loads live data |
| `PortFittingOverview.trex` | Tableau manifest — tells Tableau where to load the extension from |
| `tableau.extensions.1.latest.js` | Official Tableau Extensions API library (local copy) |

## What the worksheet must look like

The extension expects **one worksheet**, at transaction-detail grain — every row below
placed on the **Detail** shelf so each mark equals one original sales line item (same
grain as `Data from ds.xlsx`). Field names must match the source Excel headers **exactly**:

```
Day of Time Date       Sls Grp Desc          Sls Ofc Desc
Article Id              Article Name Th       Brand
Mch1 Desc                Mc Desc               Flag_PrivateBrand
Vendor Name              Col Name              flag_focus_new (group)
Class price              Net Inc Tax           Sale Qty
%Margin (no tax)
```

If any of these are missing or misnamed, the extension shows an error banner naming
the missing field(s) instead of a blank/broken dashboard.

Product Group (Faucet/Shower/Spare Parts/Accessories) and Color Group (Black/Gold/
Other/Rose Gold/Silver/White) are derived in-browser from `Mch1 Desc` / `Col Name`
using the same mapping tables as `Data mapping.xlsx` — no extra worksheet fields needed
for those two.

## 1. Push this folder to GitHub

```
cd extension
git init
git remote add origin https://github.com/warinda-nor/port_product_mer.git
git add .
git commit -m "Add Port Fitting Overview Tableau extension"
git branch -M main
git push -u origin main
```

(If the repo already has other content, copy just these 3 files into it instead of
running `git init` again.)

## 2. Enable GitHub Pages

Repo → **Settings → Pages** → Source: **Deploy from a branch** → Branch: **main**, folder **/ (root)** → Save.

Pages will publish at:
```
https://warinda-nor.github.io/port_product_mer/index.html
```
This must match the `<url>` inside `PortFittingOverview.trex` — it already does, but
double-check after Pages finishes its first deploy (can take a minute or two).

## 3. Build the worksheet + add the extension in Tableau Desktop

1. Connect the workbook to the real data source (replacing `Data from ds.xlsx`).
2. Build one worksheet with all the fields listed above on the **Detail** shelf (no
   aggregation, no filters that change the grain — actual dashboard filters are fine,
   see below).
3. Add that worksheet to a dashboard, then **Objects → Extension** → browse to
   `PortFittingOverview.trex` → **Add**.
4. The dashboard should populate within a few seconds. Any native Tableau filter you
   place on the dashboard (Year, Zone, Office, Channel, Vendor, Brand, …) will
   automatically re-trigger the extension and refresh every chart — that's what the
   `FilterChanged`/`SummaryDataChanged` listeners in `index.html` are for.

## Local testing (optional, before deploying to GitHub Pages)

Tableau extensions must be served over http(s), not opened as a local `file://` path.
To test against `http://localhost` before pushing:

```
cd extension
node -e "require('http').createServer((req,res)=>{const fs=require('fs'),path=require('path');let p=path.join(__dirname,decodeURIComponent(req.url.split('?')[0])==='/'?'/index.html':decodeURIComponent(req.url.split('?')[0]));fs.readFile(p,(e,d)=>{if(e){res.writeHead(404);res.end('not found');return;}res.writeHead(200);res.end(d);});}).listen(8765,()=>console.log('http://localhost:8765'))"
```

Then temporarily point the `<url>` in a copy of `PortFittingOverview.trex` at
`http://localhost:8765/index.html`, load that manifest in Tableau Desktop, and swap
back to the GitHub Pages URL once confirmed working.

## What still needs your input

- Nothing left on my end for the extension itself — it's ready as soon as the real
  worksheet exists and the repo is pushed.
- If field names in the real workbook end up differing even slightly (extra spaces,
  renamed during import, etc.), the extension will fail loudly with a named-field
  error rather than silently showing wrong numbers — rename the Tableau field (or the
  constant in `index.html`'s `FIELD` object) to match.
