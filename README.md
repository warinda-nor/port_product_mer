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

## What the worksheets must look like

A single transaction-detail worksheet (Article Id x day, ~700k rows) hits Tableau's
Extensions API row limits and fails with a generic `internal-error`. To keep both
worksheets small, the dashboard is split into **2 worksheets**, auto-detected by
whichever one has `Article Id` on Detail:

**1. Trend worksheet** — Month-grain, **no `Article Id`**. Feeds the 4 KPI tiles, the
monthly Net Sales trend chart, and Office / Private Brand / Focus (none of these need
a distinct-SKU count, so keeping them off the Detail worksheet avoids multiplying them
against thousands of SKUs).

```
Time Date               Net Inc Tax           Sale Qty
%Margin (no tax)         Sls Ofc Desc          Flag_PrivateBrand
flag_focus_new (group)
```
Rows shelf: `MONTH(Time Date)` (or `Time Date` at day grain — either works, since this
worksheet only aggregates by month). No `Article Id`, `Article Name Th`, `Mch1 Desc`,
`Mc Desc`, `Vendor Name`, `Sls Grp Desc`, `Class price`, or `Col Name` on Detail —
that's what keeps this worksheet small.

**2. Detail worksheet** — **Year-grain only** (no Month/Day), **with `Article Id`**.
Feeds the breakdowns that need a distinct-SKU count — **Channel, Product Group, Price
Segment, Vendor** — plus Color Group, Top Brand, and Hero Products. Color Group lives
here (not on Trend) because `Col Name` doesn't need a SKU count either, but is simplest
to keep alongside the other Article-level attributes already on this worksheet.

```
Time Date               Article Id            Article Name Th
Brand                    Mch1 Desc             Mc Desc
Vendor Name              Sls Grp Desc          Class price
Col Name                 Net Inc Tax           Sale Qty
```
Rows shelf: `YEAR(Time Date)` — do **not** add Month/Day here, since crossing
`Article Id` with a finer date grain multiplies row count back up (this was the
original cause of the `internal-error`). Also keep `Sls Ofc Desc`, `Flag_PrivateBrand`,
and `flag_focus_new (group)` **off** this worksheet — they belong on Trend only;
adding them here re-multiplies `Article Id` against extra dimensions for no benefit,
since none of them need a SKU count.

Field names must match the source Excel headers **exactly** on both worksheets. If
any expected field is missing or misnamed on either one, the extension shows an error
banner naming the missing field(s) and which worksheet, instead of a blank/broken
dashboard. Both worksheets must be placed on the same dashboard alongside the
Extension object — the extension auto-detects which is which by checking for
`Article Id`, so it doesn't matter what you name the worksheets themselves.

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

## 3. Build the worksheets + add the extension in Tableau Desktop

1. Connect the workbook to the real data source (replacing `Data from ds.xlsx`).
2. Build the **Trend worksheet**: `MONTH(Time Date)` (or `Time Date`) plus
   `Net Inc Tax`, `Sale Qty`, `%Margin (no tax)`, `Sls Ofc Desc`, `Flag_PrivateBrand`,
   `flag_focus_new (group)` on the **Detail** shelf. No `Article Id`, no `Col Name`.
3. Build the **Detail worksheet**: `YEAR(Time Date)` plus `Article Id`,
   `Article Name Th`, `Brand`, `Mch1 Desc`, `Mc Desc`, `Vendor Name`, `Sls Grp Desc`,
   `Class price`, `Col Name`, `Net Inc Tax`, `Sale Qty` on the **Detail** shelf. No
   `Sls Ofc Desc`, `Flag_PrivateBrand`, or `flag_focus_new (group)` here.
4. Add **both** worksheets to the same dashboard, then **Objects → Extension** →
   browse to `PortFittingOverview.trex` → **Add**.
5. The dashboard should populate within a few seconds. Any native Tableau filter you
   place on the dashboard (Year, Zone, Office, Channel, Vendor, Brand, …) will
   automatically re-trigger the extension and refresh every chart — that's what the
   `FilterChanged`/`SummaryDataChanged` listeners in `index.html` are for (registered
   on both worksheets).

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
