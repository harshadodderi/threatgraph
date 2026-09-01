# IOC Trend

Client-side viewer for AlienVault OTX IOC exports. Drop in a CSV or XLSX, get an IOC-count-vs-date chart, download it as a PNG.

Everything runs in the browser tab. The file is read with the `FileReader` API and never transmitted — there is no backend, no upload endpoint, and no telemetry.

**Live:** https://harshadodderi.github.io/threatgraph/

---

## Why

The manual version of this was an Excel pivot: export IOCs from OTX, pivot `Date` × `Type`, merge the three hash columns by hand, drop the chart into a slide. That is ten minutes of clicking every time the export changes, and the hash-merge step gets forgotten. This does the same thing in one drop, and the output is deterministic.

Companion to [multi-ioc-exporter](https://github.com/harshadodderi/multi-ioc-exporter), which produces the exports this reads.

---

## Use

1. Open the page.
2. Drop the export onto the panel, or click to browse. `.csv`, `.tsv`, `.xlsx`, `.xls`.
3. Adjust bucket / range / display.
4. **Download PNG** for the chart, **Download summary CSV** for the pivoted counts.

### Input format

Four columns are used. Everything else in the file is ignored.

| Column | Required | Notes |
|---|---|---|
| `Date` | yes | `YYYY-MM-DD`, ISO timestamp, `M/D/YYYY`, or an Excel date serial |
| `Type` | yes | Raw OTX type strings — normalised, see below |
| `IOC Value` | no | Only used when **Dedupe IOCs** is on — the dedup key |
| `Pulse` | no | Distinct values per bucket become the *Intel reports* line |

Headers are auto-detected, case-insensitively, with fuzzy fallback (`Created` will be picked up if there is no `Date`, `Indicator Type` if there is no `Type`). If the guess is wrong, open **Column mapping** in the sidebar and set them yourself.

The straight OTX export header — `Date, IOC Value, Type, Pulse, Pulse URL, Author, TLP, Tags, Created, Description` — works with no mapping.

### Type normalisation

| Raw value | Charted as |
|---|---|
| `FILEHASH-MD5`, `FILEHASH-SHA1`, `FILEHASH-SHA256` | **Hashes** (one series) |
| `IPV4`, `IPV6`, `CIDR` | **IP** |
| `DOMAIN` / `HOSTNAME` | kept separate — OTX distinguishes them |
| `URI`, `URL` | **URL** |
| `CVE`, `EMAIL`, `YARA`, `BITCOINADDRESS`, `FILEPATH`, `MUTEX` | as labelled |
| anything unrecognised | **Other** |

Turn off **Merge hash types** to split MD5 / SHA1 / SHA256 into their own series.

---

## Controls

| Control | What it does |
|---|---|
| Bucket | Day, week (day-of-month blocks — see below), or month |
| Bars | Grouped or stacked |
| Merge hash types | One `Hashes` series vs. three |
| Dedupe IOCs (first seen) | Collapse repeated indicators to one row on their earliest date — see below |
| Print values on bars | Numeric labels, as in the original Excel chart |
| Intel count line | Distinct `Pulse` values per bucket, plotted on the right axis |
| Logarithmic axis | For exports where one type dwarfs the rest |
| From / To | Date range; `Reset range` restores the full span |
| Column mapping | Manual override when auto-detection misses |
| Background | Colour of the exported PNG canvas. `Dark` / `White` presets, or any colour via the picker |
| Aspect ratio | 16:9, 4:3, 3:2, 1:1, or a custom W×H. Sizes the canvas, so the PNG comes out at that ratio |
| Resolution | Export multiplier — 1x, 2x, 3x. Exact output dimensions are shown beneath the selector |

The legend is interactive — click a type to drop it from the chart. That state is captured in the PNG.

### Week bucketing

Week buckets are **fixed day-of-month blocks**, not ISO calendar weeks: days 1–7, 8–14, 15–21, and 22–end. The final block absorbs the tail of the month, so February ends at 22–28 and a 31-day month at 22–31 — there is no stray one-to-three-day bucket.

Blocks never cross a month boundary. A run of dates spanning the end of July into August splits into a July 22–31 block and an August 01–07 block; no August week ever counts July days. This is deliberate: it makes "week 2" mean the same span (the 8th–14th) in every month, so the same block is comparable across months regardless of which weekday the month started on. Labels read `Aug 01–07`, `Aug 08–14`, and so on.

If you need true Monday-start ISO weeks, this is not that — the trade-off is that these blocks do not align to calendar weeks, which matters if you reconcile against a SIEM's ISO weekly rollup.

### Deduping indicators

By default the chart is a **count of sightings**, not unique indicators: the same IOC reported by three pulses on three dates counts three times. That is often what a hunting trend wants — it reflects reporting intensity — and it matches the Excel chart this replaces.

Turn on **Dedupe IOCs (first seen)** to count each distinct indicator once instead. The dedup key is `(IOC Value, Type)` and the earliest date wins, so a hash flagged by nine pulses across a week collapses to a single row on the day it first appeared. `Type` is part of the key, so the same string classified two ways is not merged, and dedup uses the raw type — it is independent of the **Merge hash types** toggle. Rows with no `IOC Value` are never dropped; they pass through unchanged. When dedup is on, the readout's last cell switches to **Dupes removed** and shows how many rows collapsed.

Dedup applies only to the bars. The **Intel reports** line always counts distinct pulses per bucket, so with dedup on a bucket can show fewer IOC bars than the pulse line implies — the two axes measure different things and should not be read as tracking each other.

### PNG background and ratio

The canvas is painted opaque before export, so the PNG is never transparent. The chart's text, gridlines and axis colours are chosen from the **relative luminance** of the background you pick — set a light background and the ink flips dark automatically, so a white export for a slide deck stays legible without any further tweaking. This applies to the on-screen chart too, so what you see is what the file contains.

Chart typography is **derived from the canvas size**, not fixed in absolute pixels. The base unit tracks the geometric mean of the plot box, so the title holds around 3.5% of frame height and tick labels around 2.4% at every ratio and resolution. Without this, a large export renders correct-but-microscopic text — a 4121×2795 PNG with 10px ticks puts the labels at 1% of frame height, which is unreadable at fit-to-screen.

If the requested ratio needs more height than the viewport allows, the **width** shrinks to compensate. The exported PNG is always the ratio you selected; it is never silently clipped to something else.

`Resolution` sets the export multiplier (1× / 2× / 3×). The sidebar shows the exact output dimensions before you click.

### Dynamic range

IOC exports are heavily skewed — a single `Hashes` bucket can be several hundred while `Bitcoin address` is 1. On a linear axis the small series are not merely hard to read, they are geometrically absent: at a 391:1 spread the smallest bar occupies 0.26% of the plot height, roughly 5px in a 2795px-tall export. No amount of resolution fixes this, because the bar is that size by construction.

When the spread across the visible buckets exceeds 40:1, a notice appears above the chart stating the actual ratio and offering a one-click switch to the log axis. The threshold is a heuristic, not a rule — the notice never changes the chart on its own.

**Undated rows** in the readout is a data-quality signal: it counts rows whose `Date` could not be parsed at all. If it is not zero, the wrong column is mapped or the export is malformed. (With **Dedupe IOCs** on, this cell is replaced by **Dupes removed**.)

---

## Dates

Dates are normalised in **UTC end to end** — Excel serials, ISO strings, and slash dates all resolve to a `YYYY-MM-DD` day without passing through local time. This matters because a local-time round-trip can shift a date across midnight, which in turn moves it into the wrong day, week block, or month for anyone east or west of UTC. If your export looked like it was pulling the previous month's data, that drift was the cause, and it is fixed.

Slash dates are still read as `M/D/YYYY` (see Known limits). ISO dates are unambiguous — prefer them.

---

## Deploy

Single file, no build step.

```bash
git init
git add index.html README.md
git commit -m "IOC trend viewer"
git remote add origin https://github.com/harshadodderi/threatgraph.git
git push -u origin main
```

Then **Settings → Pages → Source: Deploy from a branch → `main` / `root`**.

If you deploy via GitHub Actions instead (as `multi-ioc-exporter` does), use the `static.yml` starter workflow and set **Source: GitHub Actions**.

---

## Dependencies

Loaded from CDN at page load; nothing is called afterwards.

| Library | Version | Role |
|---|---|---|
| PapaParse | 5.4.1 | CSV / TSV parsing, in a worker |
| SheetJS (`xlsx`) | 0.18.5 | XLSX / XLS reading |
| Chart.js | 4.4.1 | Rendering |
| chartjs-plugin-datalabels | 2.2.0 | Bar value labels |

The page needs internet access on first load. To run it fully offline, download the four `.min.js` files into `lib/` and repoint the four `<script src>` tags.

SheetJS left npm and cdnjs after 0.18.5. If the cdnjs copy disappears, the page falls back to `cdn.sheetjs.com` automatically on the first XLSX file. CSV is unaffected either way.

---

## Known limits

- **Slash dates are read as `M/D/YYYY`.** A `D/M/YYYY` export will swap day and month for the 1st–12th. ISO dates are unambiguous — prefer them.
- **The chart is a count by default, not a dedup.** The same indicator appearing in three pulses on one day counts three times. That matches the Excel chart it replaces. Turn on **Dedupe IOCs (first seen)** to count unique `(IOC Value, Type)` pairs instead.
- **Weeks are day-of-month blocks, not ISO calendar weeks.** 1–7, 8–14, 15–21, 22–end; they never cross a month. If you need Monday-start ISO weeks, this is not that.
- **Large files.** ~3k rows renders instantly; tens of thousands will still parse but the day-bucketed chart gets unreadable — switch to week or month.
- **PNG export** is the canvas at the chosen resolution multiplier, with an opaque background so it drops straight into a deck. It is raster only, no SVG.

---

## Privacy

No `fetch` or `XMLHttpRequest` to any endpoint after the initial script loads. No cookies, no `localStorage`, no analytics. Confirm it yourself: open DevTools → Network, load a file, and watch nothing happen.

That said, the four CDN requests at page load do reveal your IP and referrer to Cloudflare. Vendor the libraries locally if that matters for your threat model.

---

## Author

Harsha Dodderi — [LinkedIn](https://www.linkedin.com/in/harshadodderi/) · [GitHub](https://github.com/harshadodderi)
