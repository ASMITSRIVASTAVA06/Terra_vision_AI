# 🌿 TerraVision AI — Forest Encroachment Detection

Satellite-powered detection of illegal forest encroachment using NASA VIIRS night-light
data, cross-checked against other satellites, with a review workflow for field teams.

## What it does
1. **Detects** clusters of new night light by comparing the newest VIIRS months against a 5-year baseline.
2. **Classifies** each cluster (illegal encroachment, wildfire, agricultural burn, anomaly) with a Random Forest.
3. **Cross-checks** every alert against independent data and adjusts its confidence (see below).
4. **Tracks** alerts across scans: new, recurring, or resolved.
5. **Routes** alerts to people: review status, assignee, notes, e-mail, Telegram, WhatsApp, scheduled digests.

## The evidence checks
| Check | Dataset | Effect |
|---|---|---|
| Protected area | WDPA polygons | inside: + · outside: − |
| Urban light | ESA WorldCover (built-up share of the pixel) | ≥ 30 % built-up: strongly −, flagged "Urban light" |
| Vegetation change | Sentinel-2 NDVI, last 90 days vs same season last year | drop: + |
| Forest loss | Hansen Global Forest Change | recent loss: + |
| Active fire | NASA FIRMS thermal anomalies | confirms fires, doubts "settlements" |
| Persistence | seen in 2+ satellite periods | + for encroachment |

Every rule and weight is listed in `evidence.py`. The model's own confidence is always kept
(`model_confidence`) next to the adjusted one. Each alert gets a verdict: *Strong evidence*,
*Possible* or *Likely false positive*. If a dataset is unavailable the scan still completes and
tells you which checks were skipped.

## App sections
- **Monitor** — scan a region and see the alerts on the map (satellite-imagery layer included).
- **Dashboard** — interactive charts: alert mix, severity, evidence verdicts, new vs recurring, signals behind the
  alerts, confidence shift, scan-to-scan trend, distributions, map, classifier insight.
- **Alerts** — filter and select alerts, record a review (status / assignee / note / confirmed type), see the exact
  location on satellite imagery with the 500 m pixel outlined, then download (CSV, JSON, printable report),
  e-mail, send to Telegram or WhatsApp, or export reviewed alerts as training labels.

Only real scan results are shown. Every scan is saved to a local SQLite database (`terravision.db`).

## How to run

```bash
pip install -r requirements.txt
cp .env.example .env            # then fill in GEE_PROJECT_ID (and optional SMTP / Telegram settings)
python test_connection.py       # one-off Earth Engine login check
streamlit run app.py
```
Press **Run Satellite Scan** in the Monitor section. The first scan of a region downloads the baseline
and latest data from Earth Engine (a few minutes); later scans reuse the cache in `data_cache/`.
The "current" period always rolls forward to the newest VIIRS month available.

## Fewer false alerts: regional brightening correction
Villages get electrified, so whole districts glow a little brighter each year and "now minus 5-year median" is positive
almost everywhere. `regional.py` estimates that smooth background trend (block medians, about 8 km) and removes it, so
the detector looks for patches that stand out from their own surroundings. Only brightening is removed; a region
getting darker never creates alerts. It is on by default (sidebar: *Correct for regional brightening*, or
`--no-regional-correction` for the scheduler). Expect fewer alerts than before; compare a scan with and without it.
Scans remember the setting, and a scan is only compared with an earlier one made with the same setting.

## Look at the ground: before / after imagery
In the Alerts section (and the Dashboard map), pick an alert and press *Show before / after satellite imagery*.
You get two Sentinel-2 true-colour views of the same place and season one year apart, with the 500 m satellite pixel
outlined in yellow. It is fetched on demand, one alert at a time.

## How trustworthy are the confidence scores?
After you have reviewed alerts (verified / false positive), run:
```bash
python calibrate.py                                   # reads terravision.db
python calibrate.py --json terravision_alerts.json    # or an export from the Alerts section
```
It reports precision by evidence verdict, claimed-versus-actual accuracy per confidence band, Brier score and
calibration error for the model alone versus the evidence-adjusted confidence, and (with about 60+ labels) a
cross-validated suggestion for re-weighting the evidence signals. Review alerts across *all* confidence levels:
a sample of only the top alerts makes everything look better than it is.

## Reliability and operations
- **Retries:** transient Earth Engine errors (quota, timeouts, 503) are retried with exponential backoff; permanent
  errors (missing asset, permissions) fail at once with a clear message.
- **Run history:** every scan attempt, successful or failed, is recorded. See *Scan history & health* in the Monitor
  section; details are also in `logs/terravision.log`.
- **Audit trail:** every review change (who, when, old and new status, note) is kept and shown for the selected alert.
  Set `TERRAVISION_USER` to record a name other than the operating-system user.
- **Database versions:** `terravision.db` upgrades itself in place when the app is updated.
- **CI:** `.github/workflows/tests.yml` runs the tests on every push.

## Improving the classifier with real labels
The shipped model was trained on synthetic data. Mark alerts **verified** (or **false positive** with the
true type) in the Alerts section, download `terravision_labels.csv` from the *Training labels* tab, then:
```bash
python model.py --csv terravision_labels.csv
```

## Scheduled scans and digests
`scheduled_scan.py` runs the same pipeline headless and sends a digest only when something **new**, credible
and still unreviewed appears (inside a protected area, not urban, confidence above the bar).
```bash
python scheduled_scan.py --regions all --telegram --to "ranger@example.org"
python scheduled_scan.py --regions "Aravalli Hills, India" --dry-run      # print, send nothing
```
Run it from cron, or use `.github/workflows/scheduled_scan.yml` (needs repository secrets listed in the file;
the database is kept in the Actions cache, which is dropped after 7 idle days).

## Tests
```bash
pip install -r requirements-dev.txt
python -m pytest
```
The tests cover everything that needs no satellite access: alert tracking, evidence scoring, storage and migrations,
exports, digests, retries, the rolling window, regional correction and calibration. Earth Engine cross-checks and the Streamlit pages are not covered by
automated tests; check them by running a scan.

## Project structure
- `app.py` — Streamlit app (navigation + Monitor)
- `dashboard.py`, `alerts_view.py`, `locator.py` — Dashboard, Alerts and exact-location map
- `pipeline.py` — scan pipeline shared by the app and scheduler
- `detect.py`, `model.py` — change detection and classifier
- `enrichment.py`, `evidence.py` — satellite cross-checks and evidence scoring
- `tracking.py`, `store.py` — alert tracking across scans, SQLite storage and review state
- `alert_tools.py` — exports, e-mail, Telegram, training labels
- `regional.py` — regional brightening correction · `imagery.py` — before/after imagery
- `calibrate.py` — calibration report from reviewed alerts · `reliability.py` — retries and logging
- `scheduled_scan.py` — headless scan and digest
- `regions.py` — supported regions · `gee_export.py` — optional manual raster export · `tests/`

## Data sources
NASA VIIRS DNB (`NOAA/VIIRS/DNB/MONTHLY_V1/VCMSLCFG`) · WDPA · ESA WorldCover · Sentinel-2 SR ·
Hansen Global Forest Change · NASA FIRMS

## Limits worth knowing
VIIRS pixels are ~500 m: a pin marks the centre of a glowing area, not a building. Night-light change
suggests, it does not prove — verify every alert on the ground. Earth Engine dataset IDs and quotas change over
time; the cross-check layers degrade gracefully if one is unavailable.

## Team
EcoSentinel — YESIST12 Innovation Challenge 2026


## Real boundaries, reliable data and ML training

**Region boundaries.** Each region is no longer just a rectangle. The rectangle is only the
download box (drawn dashed on the map). The real footprint is, in order of trust:
1. an official GeoJSON you place in `boundaries/<region_slug>.geojson` (forest department, FSI,
   Survey of India, Protected Planet, OpenStreetMap), or
2. designated terrestrial protected areas from WDPA, restricted to the region's search box and its
   countries, cached in `data_cache/boundaries/` and refreshed every 90 days.
Alerts are tested against these polygons locally (holes and multipolygons respected) and get a
zone: official / protected / forest / outside. "Forest & protected areas only" hides *outside*.
Region names → file names: see `regions.region_slug`.

**Reliable satellite data.** VIIRS now downloads the cloud-free observation count (`cf_cvg`)
with the radiance, and pixels seen cloud-free fewer than twice are ignored. Old single-band
files still load but cannot be quality-checked, so the baseline is rebuilt once (`*_q2.tif`).
WDPA checks use only designated areas; land cover adds a 1 km forest/cropland context.

**ML on real labels.**
```
python training_data.py --regions all --windows 6   # Earth Engine; resumable; writes training/real_dataset.csv
python train_real.py                                # spatial CV, calibration, quality gate
```
Labels come from satellites the classifier never sees (FIRMS fire, WorldCover, Hansen loss,
NDVI, WDPA); ambiguous clusters are dropped. Alerts you verify in the app are merged with
double weight (`--db terravision.db`). The new model is saved only if it clears a macro-F1 gate
and beats the current model on the same real labels (`--force` overrides). `model_card.json`
records what the classifier was trained on, and the sidebar shows it.
These labels are weak supervision, not ground truth: review alerts to improve them.

## Look and feel ("Abyss")

`theme.py` holds the whole design system: deep ocean-black with an electric-blue to cyan gradient,
frosted-glass panels, a radar hero, gauge rings, badges and a light variant (toggle top right).
Alert colours are semantic and never blue: red encroachment, orange wildfire, amber agricultural
burn, steel anomaly. Change `DARK` / `LIGHT` in `theme.py` to re-colour everything, including the
charts; `.streamlit/config.toml` themes the native widgets to match.

## Weather and cyclone risk

The **Weather risk** section forecasts where thunderstorms, heavy rain, damaging wind, extreme heat
and tropical cyclones may hit, and flags the risky zones on the map.

* **Forecast:** Open-Meteo (free, no key). Hourly rain, gusts, temperature, storm energy (CAPE) and
  weather codes are scored per map cell over the next 24–120 h: Low / Watch / Warning / Severe.
  Rain classes follow IMD's 24 h thresholds (heavy 64.5 mm, very heavy 115.6 mm, extremely heavy
  204.5 mm); gusts from 118 km/h count as hurricane force. Storm energy alone is only a Watch; a
  thunderstorm must also appear in the forecast to reach Warning.
* **Cyclones:** active storms, with observed and forecast tracks and gale-force wind fields, come
  from GDACS (which republishes NOAA / JTWC advisories). A zone is flagged when it lies inside the
  forecast wind field or near the forecast track.
* **Where:** any forest region, or any place you search, at coarse / medium / fine detail. "All forest
  regions" checks every monitored region at once.
* **Alerts:** the "Send alert" tab, or unattended: `python scheduled_weather.py --regions all
  --min-level 2 [--telegram] [--to a@b.org] [--dry-run]`. It only sends when something at or above
  the level appears and has changed since the last alert. Schedule it with cron
  (`15 */6 * * *`) or the included `.github/workflows/scheduled_weather.yml`.
* **Honesty rules:** a zone whose forecast could not be fetched is shown as unknown, never as safe;
  if the cyclone feed is down the page says so instead of reporting "no cyclones".

These are indications from model forecasts and public advisories, not official warnings. Follow IMD,
NHC, JTWC or your national weather service for decisions that affect safety.

## Connecting real data (API keys)

TerraVision never shows sample data: every number comes from a live service. Put your keys in `.env`
(copy `.env.example`), or in Streamlit secrets when hosting, then run:

```
python check_apis.py     # one tiny request per service; prints OK / FAILED, never prints a key
```

The sidebar's **Data sources** panel shows the same status (green key set, yellow free service, red key missing).

| Service | Key variable | What it adds | Cost | Where to get it |
|---|---|---|---|---|
| NASA FIRMS | `FIRMS_MAP_KEY` | Real fire hotspots, usually within hours of the satellite pass (VIIRS S-NPP and NOAA-20). Used by place search; Earth Engine's daily FIRMS layer is the fallback. | Free | firms.modaps.eosdis.nasa.gov/api/map_key |
| Google Earth Engine | `GEE_PROJECT_ID` (+ `GEE_SERVICE_ACCOUNT_JSON` when hosting) | Night lights, protected areas, land cover, forest loss, vegetation | Free for research | earthengine.google.com |
| Open-Meteo | `OPENMETEO_API_KEY` (optional) | Weather forecast. Works with no key for non-commercial use; a paid key is needed for commercial use and switches to the paid host automatically | Free / paid | open-meteo.com/en/pricing |
| Geocoder | `GEOCODER_API_KEY`, `GEOCODER_URL` (optional) | Place search at production volume. The free OpenStreetMap service is for light use; any Nominatim-compatible service works, e.g. LocationIQ | Free tier | locationiq.com |
| GDACS | none | Active cyclones and forecast tracks | Free | gdacs.org |

Recommended order: **FIRMS first** (free, takes two minutes, biggest gain in real-time accuracy), then
Earth Engine if not already set up, then a geocoder key if several people will use the place search.
Alerts use your existing `SMTP_*` and `TELEGRAM_*` settings.

Keys are read from the environment only, are never logged, and are stripped from any error text
before it reaches the screen. Keep `.env` out of version control (it is already in `.gitignore`)
and out of any zip you share.
