# UK Winter Wheat — In-Season Area & Yield Forecast

Static demo portal: weekly, field-level winter-wheat area and yield forecasts for the
Great Britain arable belt. Runs entirely on GitHub Pages — no server, no object store,
no runtime dependency on the processing cluster.

## What is here

| path | what | ships |
|---|---|---|
| `index.html` | viewer (MapLibre GL + PMTiles) | once |
| `lib/` | vendored maplibre-gl 5.6.0 + pmtiles (no CDN) | once |
| `tiles/fields.pmtiles` | field geometry, one feature per parcel | **once** |
| `values/<week>.bin` | per-week `uint8` wheat flag + `uint8` yield | per week |
| `admin/*.geojson` | ONS boundaries (country/region/county/LAD) | once |
| `manifest.json` | weeks, KPIs, admin statistics, method text | per week |

Geometry is shipped **once** because the 5 m field segmentation is static across the
season. Only the values change weekly, so adding a week costs ~450 kB rather than
re-publishing the map.

Yield is encoded as `uint8` over 0–20 t/ha (`value * 20 / 254`) — the same convention the
original display COGs used, so numbers are directly comparable. Quantisation RMSE is
0.023 t/ha, about 15× finer than the model's own systematic uncertainty (0.35 t/ha).

## Deploying

1. Create a repo and push this directory to the default branch.
2. **Settings → Pages → Source: Deploy from a branch**, pick the branch and `/ (root)`.
3. The `CNAME` file already carries `wheat.rayx.co.uk`. It is committed rather than
   set only in the Pages UI, so a force-push cannot silently drop it.
4. At your DNS provider add a `CNAME` record for that subdomain pointing at
   `<user>.github.io`. If DNS is on Cloudflare, set it to **DNS only (grey cloud)**
   until GitHub has issued the certificate, or the ACME challenge can fail.
5. Back in Settings → Pages, tick **Enforce HTTPS** once the certificate appears.

`.nojekyll` is present so Pages serves the files verbatim instead of running Jekyll.

## Why this fits on GitHub Pages

GitHub Pages caps a published site at **1 GB** and blocks any single file over
**100 MB**; Git LFS does not help, because Pages serves the LFS *pointer* rather than the
content. Rasterising this product to image tiles would need ~0.5–0.8 GB **per week**.
As vectors the whole season is a small fraction of the budget, and PMTiles works because
Pages honours HTTP range requests (verified: `206` with a correct `content-range`).

## Regenerating

```bash
# 1. per-tile vectorisation (SLURM array, one task per MGRS tile)
sbatch --array=0-42 vec.sbatch

# 2. assemble the pack
python scripts/build_portal_pack.py \
    --vec <vec-dir> --out <pack-dir> \
    --weeks 2026-07-13 2026-07-20 2026-07-27 2026-08-03
```

Seam handling reuses the processing pipeline's own `uk_mosaic.DataDepth`, so each field is
attributed to exactly the tile that owns it in the published headline. Without it the belt
would double-count the ~29–47 % of overlapping pixels along MGRS seams.

## Caveats carried from the source product

- Area is a direct pixel count with **no confusion-matrix adjustment**, so there is no area
  confidence interval. The error is classification error, not sampling error.
- Polygon parts under 500 m² are dropped as classification speckle; see `pack_report.json`
  for the exact hectares this removes.
- A week processed during a cloud gap under-reports and steps up at the next clear window.
  Per-week observation support is shown beneath the map.

## Attribution

Produced by University College London (UCL) and the National Centre for Earth
Observation (NCEO), with the UK Centre for Ecology & Hydrology (UKCEH), as part
of the AgZero+ project.

Contains modified Copernicus Sentinel-2 data (2019–2026). Administrative
boundaries © Office for National Statistics, Open Government Licence v3.0.
Basemaps © OpenStreetMap contributors © CARTO; satellite imagery © Esri.
Forecasts are research output and are provided without warranty.
