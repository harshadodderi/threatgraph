# IOC Trend

Client-side viewer for AlienVault OTX IOC exports. Drop in a CSV or XLSX, get an IOC-count-vs-date chart, download it as a PNG.

Everything runs in the browser tab. The file is read with the `FileReader` API and never transmitted — there is no backend, no upload endpoint, and no telemetry.

**Live:** https://harshadodderi.github.io/ioc-trend/

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

Three columns are used. Everything else in the file is ignored.

| Column | Required | Notes |
|---|---|---|
| `Date` | yes | `YYYY-MM-DD`, ISO timestamp, `M/D/YYYY`, or an Excel date serial |
| `Type` | yes | Raw OTX type strings — normalised, see below |
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
| Bucket | Day, ISO week (Monday start), or month |
| Bars | Grouped or stacked |
| Merge hash types | One `Hashes` series vs. three |
| Print values on bars | Numeric labels, as in the original Excel chart |
| Intel count line | Distinct `Pulse` values per bucket, plotted on the right axis |
| Logarithmic axis | For exports where one type dwarfs the rest |
| From / To | Date range; `Reset range` restores the full span |
| Column mapping | Manual override when auto-detection misses |
| Background | Colour of the exported PNG canvas. `Dark` / `White` presets, or any colour via the picker |
| Aspect ratio | 16:9, 4:3, 3:2, 1:1, or a custom W×H. Sizes the canvas, so the PNG comes out at that ratio |

The legend is interactive — click a type to drop it from the chart. That state is captured in the PNG.

### PNG background and ratio

The canvas is painted opaque before export, so the PNG is never transparent. The chart's text, gridlines and axis colours are chosen from the **relative luminance** of the background you pick — set a light background and the ink flips dark automatically, so a white export for a slide deck stays legible without any further tweaking. This applies to the on-screen chart too, so what you see is what the file contains.

Aspect ratio resizes the plot container rather than scaling the image, so labels and tick text keep their real size instead of stretching. Export is at `devicePixelRatio: 2`, so a 16:9 chart in a 1200px-wide column exports at roughly 2400×1350.

**Undated rows** in the readout is a data-quality signal: it counts rows whose `Date` could not be parsed at all. If it is not zero, the wrong column is mapped or the export is malformed.

---

## Deploy

Single file, no build step.

```bash
git init
git add index.html README.md
git commit -m "IOC trend viewer"
git remote add origin https://github.com/harshadodderi/ioc-trend.git
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
- **The chart is a count, not a dedup.** The same indicator appearing in three pulses on one day counts three times. That matches the Excel chart it replaces; if you need unique indicators, dedup on `IOC Value` before loading.
- **Weeks start Monday** (ISO 8601), regardless of locale.
- **Large files.** ~3k rows renders instantly; tens of thousands will still parse but the day-bucketed chart gets unreadable — switch to week or month.
- **PNG export** is the canvas at 2× device-pixel ratio, with an opaque background so it drops straight into a deck. It is raster only, no SVG.

---

## Privacy

No `fetch` or `XMLHttpRequest` to any endpoint after the initial script loads. No cookies, no `localStorage`, no analytics. Confirm it yourself: open DevTools → Network, load a file, and watch nothing happen.

That said, the four CDN requests at page load do reveal your IP and referrer to Cloudflare. Vendor the libraries locally if that matters for your threat model.

---

## Author

Harsha Dodderi — [LinkedIn](https://www.linkedin.com/in/harshadodderi/) · [GitHub](https://github.com/harshadodderi)
