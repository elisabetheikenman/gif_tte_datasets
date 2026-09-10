# New TTE datasets — what we can use, and what it costs

Two passes: the three sources sent first (Quebec, Rome, San Francisco), then a
full sweep of `TTE_datasets_appendix.md`. Checked 27 Aug 2026.

**Already in the table, skipped throughout:** Omsk, Abakan, Chengdu, Harbin, Porto.

## What "usable" means here

The gold contract is the Harbin format with the Omsk file naming (both in
`datasets.zip`):

| file | contents |
|---|---|
| `matched_trips_<city>.csv` | unnamed index, `Id`, `Coordinates`, `OSMids`, `Timestamps`, `Total_time` |
| `edge_list_directed_<city>.csv` | `osmid_u`, `osmid_v` — directed segment-to-segment transitions |
| `road_network_unique_osmids_<city>.geojson` | one `LineString` per segment + `osmid` / `original_osmid` / `is_duplicate` / `duplicate_index` / `length` / `highway` |

`Coordinates` / `OSMids` / `Timestamps` are equal-length Python-literal lists, one
entry per GPS fix, and `Total_time = Timestamps[-1] - Timestamps[0]`.

A source is **R1 (route-aware)** only if it gives a sequence of road segments with
timestamps — already matched, or raw GPS dense enough to match ourselves.
A source that gives only origin, destination and duration is **OD-only**: usable,
but for a different task.

## Verdict summary

| # | Dataset | Mode | Verdict | Notebook |
|---|---|---|---|---|
| 1 | **eVED** (Ann Arbor) | R1 | **Best of the new ones.** ~1 Hz, on-road coordinates, explicit trips. Verified in hand | `prepare_dataset_eved.ipynb` |
| 2 | **Quebec City** | R1 | **Use.** Already map-matched. No geometry exists | `prepare_dataset_quebec.ipynb` |
| 3 | **San Francisco** Cabspotting | R1 / OD | **Conditional.** Occupancy flag = observed trips, but at ~60 s sampling the route is mostly inferred — see 1.4 | `prepare_dataset_san_francisco.ipynb` |
| 4 | **Rome taxi** | R1 | **Use.** 7 s sampling, but trips must be inferred | `prepare_dataset_rome.ipynb` |
| 5 | **GeoLife** (Beijing) | R1 | **Use.** Only open source with transport-mode labels | `prepare_dataset_geolife.ipynb` |
| 6 | **pNEUMA** (Athens) | R1 | **Use as a calibration set, never as a city** | `prepare_dataset_pneuma.ipynb` |
| 7 | **SUMO** LuST / InTAS / MoST | R1, simulated | **Use as a diagnostic bench.** Only source of paired counterfactuals | `prepare_dataset_sumo.ipynb` |
| 8 | **Citi Bike** + 3 sister systems | OD | **Use, clearly labelled OD.** Huge, 13–16 years | `prepare_dataset_citibike.ipynb` |
| 9 | T-Drive | R1 | **Decide first** — probably our existing Beijing column | — |
| 10 | Chicago TNP / Taxi, NYC TLC | OD | Census-tract centroids, 15-min rounding. Different task | — |
| 11 | NGSIM | R1, micro | Would work; pNEUMA fills the same role with real lat/lon | — |
| 12 | LargeST / PeMS / METR-LA / UTD19 | R4 | Label synthesised by our own composition rule | — |
| 13 | Helsinki matrix, Valhalla/r5r | OD, engine | Routing-engine output, not measurement | — |
| 14 | Shenzhen SciDB, IEEE 7e1g-hw96 | R1 | Licence unstated / thin deposit | — |
| 15 | DiDi GAIA, Grab-Posisi, Quebec-full | R1 | **Send letters** | — |
| 16 | NPMRDS, Uber Movement, Chicago & Austin scooters, Aalborg, Q-Traffic, Astana-synthetic, MobilityBench, HuggingFace | — | Dead ends | — |

---

## 1. What I verified by hand this pass

Several appendix items were marked *[не верифицировано]*. These are now settled —
and two of them change the plan.

### eVED clones, and it is better than the appendix says

`git clone https://Datarepo@bitbucket.org/datarepo/eved_dataset.git` **works**
(1.3 GB). `data/eVED.zip` holds **54 weekly CSVs, 5.8 GB uncompressed**, 35
columns. Measured on the real files:

* **Sampling is ~1 Hz** — median 0.6–0.9 s between fixes. That is denser than
  everything else on this list except pNEUMA, and matches Grab-Posisi, which needs
  a letter and a wait.
* `Matchted Latitude[deg]` / `Matched Longitude[deg]` (the typo is in the data)
  are **non-null for 100 %** of the records checked, and sit ~2·10⁻⁵° off the raw
  fix — i.e. the calibration really did put every point on a road.
* Trips are explicit: `(VehId, Trip)`. Median duration ~370–420 s, 101+ points each.
* Ann Arbor bbox, lat 42.22–42.32, lon −83.80…−83.67.

**Correction to the appendix:** eVED does *not* give an edge-id sequence. It gives
snapped coordinates plus road attributes (speed limit with direction, elevation,
gradient, intersection / bus-stop / crossing flags). The OSM segment ids still
have to be produced — but from on-road coordinates, so matching is easy.

**Time base, which is documented nowhere:** `DayNum` is **constant within a trip** —
it is the trip start, in days since **2017-11-01**, 1-based. `Timestamp(ms)`
restarts at 0 each trip. So
`epoch = epoch(2017-11-01) + (DayNum−1)·86400 + Timestamp(ms)/1000`.
Verified across two weekly files: reconstructed range 2017-11-01 00:04 →
2018-11-10 12:10, exactly the collection window.

Licence is still unstated (VED itself is Apache-2.0). Ship the notebook, not the
derived files.

### The bikeshare S3 buckets are listable after all

The appendix could not enumerate them. Plain `curl` to `s3.amazonaws.com` works
from here and returns the XML index:

| system | objects | total | span |
|---|---|---|---|
| **Citi Bike** `tripdata` | 173 | **30.76 GB** | 2013 → 2026-07 |
| **Capital Bikeshare** | 111 | 1.67 GB | **2010** → 2026-07 |
| **Divvy** | 94 | 1.85 GB | 2013 (quarterly) → 2026-07 |
| **Bay Wheels** | 103 | 0.96 GB | 2017 → 2026-07 |

Schema confirmed by downloading a real month:
`ride_id, rideable_type, started_at, ended_at, start_station_name, start_station_id,
end_station_name, end_station_id, start_lat, start_lng, end_lat, end_lng, member_casual`
— real coordinates in the record, timestamps to the millisecond. The pre-2020
files use the older `starttime` / `start station latitude` names; the notebook
handles both.

### The SUMO route is exact, and SUMO now installs from PyPI

`pip install eclipse-sumo sumolib` brings the **binaries** (1.27.1 here), so no
system package is needed. `LuSTScenario` (449 MB, MIT) and `InTAS` (978 MB,
GPL-3.0) both clone. `lust.net.xml` carries `projParameter` (UTM 32), so
`sumolib` converts every edge shape to lon/lat.

The important part: run SUMO with `--vehroute-output.exit-times true` and it
writes, per vehicle, the **edge sequence and the exit time of every edge**:

```xml
<vehicle id="v3" depart="9.00" arrival="117.00">
  <route edges="-31500#0 --31540#1 …" exitTimes="27.00 39.00 …"/>
</vehicle>
```

That is our format with no matching, no segmentation and no inference at all. I
ran 300 vehicles on the real LuST network and converted them end to end.

**Teleports.** SUMO teleports a stuck vehicle and records the jump as if it were
part of the route. I confirmed all three vehicles SUMO logged as teleported are
caught by checking that consecutive edges are actually connected in the network —
that check is in the notebook, and `--time-to-teleport -1` should be used anyway.

### Cabspotting's sampling rate decides whether it is R1 at all

Running the San Francisco notebook rejected almost every trip. The immediate cause
was a bug of mine — the matcher allowed a vehicle to cross up to two edges between
fixes while the quality check demanded that consecutive matched segments share a
node. Those two are consistent at 7 s sampling and incompatible at 60 s, so the
check threw away trips for having the sampling rate they have. Fixed: the
transition model is now Newson & Krumm (network distance vs straight-line
distance, no assumption about how far apart the fixes are), and the matcher fills
in the edges the vehicle had to cross, so the emitted route is genuinely connected.

The more interesting result is what the fix exposes. Measured against ground truth
on a 100 m street grid, the share of the true route that map matching recovers:

| distance between fixes | true route recovered |
|---|---|
| 50 m | 100 % |
| 90 m | ~99 % |
| 150 m | ~81 % |
| 200 m | ~58 % |
| 300 m | ~55 % |
| 600 m | **~32 %** |

The axis is **distance, not time**: 150 m between fixes recovers ~81 % of the
route whether that gap took 15 s at 10 m/s or 120 s at 1.2 m/s — the measured
numbers are identical. A slow, congested fare sampled once a minute matches fine;
a fast one does not. (My first fix gated on elapsed seconds and rejected all
20 000 SF trips, matchable ones included. It now gates on metres.)

Downtown San Francisco is a regular grid, the worst case: many routes between two
points have exactly the same length, so the shortest path is a coin flip. The
notebook prints, before matching, what share of trips survive at each threshold
and the route accuracy that buys, then reports what share of emitted points are
observed rather than inferred. **If little survives, the honest use of Cabspotting
is OD mode** — section 6 of the notebook writes it — where the occupancy flag
still makes the *duration* real ground truth even though the path is not.

Rome at 7 s is unaffected; eVED and pNEUMA are far denser still.

### Blocked from this session (so still on the "check by hand" list)

`ieee-dataport.org`, `huggingface.co`, `zenodo.org`, `kaggle.com`,
`overpass-api.de`, `download.microsoft.com`, `open-traffic.epfl.ch`,
`data.transportation.gov`, `d37ci6vzurychx.cloudfront.net`, and github.com's web
UI (git clone and `raw.githubusercontent.com` work fine). So the **NYC TLC
2009–2016 schema question** — whether those delisted parquet files really carry
`pickup_longitude` — is still open, and GeoLife / pNEUMA / NGSIM had to be
described from their documentation rather than from the bytes.

---

## 2. The new datasets in detail

### 2.1 eVED (Ann Arbor) — R1, the strongest addition

383 private cars, Nov 2017 – Nov 2018, ~22 M records, ~1 Hz, already on-road.
Beyond travel time it carries fuel rate, MAF, HV battery current/SOC/voltage,
elevation, gradient and directional speed limits — the only public source where
**time and energy can be predicted jointly**.

Against it: one mid-sized American city, 383 cars, twelve months, largely the same
people driving the same commutes. Signal density is excellent, diversity is not.
This is not "a new city" in the sense the table means.

### 2.2 GeoLife 1.3 (Beijing) — R1, the only multimodal one

17 621 trajectories, 182 users, Apr 2007 – Aug 2012, 91 % logged every 1–5 s,
298.7 MB, direct download, no licence stated. **69 users labelled their
trajectories with the mode of transport**, which is the only way to ask whether a
TTE model trained on cars transfers to buses.

Against it: the mode mix is nothing like a taxi fleet — walk and bus dominate,
`car`/`taxi` are a minority; five years of trajectories against a 2026 OSM
network; mixed loggers and mixed sampling. And if our Beijing column is T-Drive,
GeoLife is a *different* Beijing — name which is which, never merge them.

### 2.3 pNEUMA (Athens) — R1, but a calibration set

10 drones over 1.3 km² of central Athens, ~100 intersections, 4 days in Oct 2018,
6 half-hour windows 08:00–11:00, **25 fps**, ~500 000 trajectories, CC BY-NC 4.0.
Format is a ragged wide CSV: 4 vehicle fields then repeating groups of
`lat, lon, speed, lon_acc, lat_acc, time`.

This is the only open source where the true continuous path of every vehicle
through a dense urban network is known. That makes it the right instrument for the
noise-floor question — how much of a segment's travel time is predictable and how
much is luck with the light phase — and for validating a per-edge composition
rule instead of postulating it.

**It must not be listed as a city.** 1.3 km², three morning hours. A track starts
when a vehicle enters the drone footprint and ends when it leaves, so `Total_time`
is a segment time, not a journey. Non-commercial licence. Rush hour only.

### 2.4 SUMO scenarios — R1, simulated, and the only counterfactuals

LuST (MIT), InTAS (GPL-3.0), MoST (GPLv3), the DLR set including TAPASCologne
(EPL-2.0). Conversion is exact (see above).

The unique value is **paired worlds**: re-run the same demand with an edge closed,
demand at ±20 %, or a different signal plan, and every vehicle has a matched
before/after. No real dataset can produce that. The notebook writes
`counterfactual_pairs_<city>.csv` when a second run is configured.

The label is a car-following model's output. LuST and InTAS are calibrated against
aggregate counts, which constrains flows and says nothing about individual trip
realism. Diagnostic bench only, and never pooled with real labels without a domain
flag.

### 2.5 Citi Bike and the three sister systems — OD, huge, noisy

Verified above. ~30.8 GB, 2013 → July 2026 for Citi Bike; Capital Bikeshare goes
back to **2010**. Duration is measured; the route is never observed.

The notebook writes the gold format so the same loader works, but each trip has
exactly **two** points, and it drops a `MODE_<city>.json` flag file next to the
data so nothing downstream mistakes these for routes. The label folds together
route choice, rider fitness, e-bike vs pedal and mid-ride detours, so the
irreducible variance is large — which is precisely what makes it a fair test of
**calibration** rather than point accuracy.

Legal: Divvy and Capital Bikeshare forbid redistributing the data as a stand-alone
dataset (CaBi adds non-commercial); Citi Bike is permissive for analysis. Publish
the loader, not the files. Operators already removed every ride under 60 s, so the
left tail of the label distribution is truncated.

---

## 3. Everything else in the appendix, and why there is no notebook

* **T-Drive** — 10 357 Beijing taxis, Feb 2008. The appendix argues our existing
  Beijing column probably *is* T-Drive. Settle that before adding anything: if it
  is, a notebook would be double-counting; if it is not, the Beijing row needs a
  documented source either way. Half a day of checking.
* **Chicago TNP (≈521 M) + Chicago Taxi (≈218 M) + NYC TLC** — the largest public
  corpora of measured durations anywhere, and the right material for scaling,
  fairness and distribution-shift work. But coordinates are census-tract or
  community-area centroids and times are rounded to 15 minutes, so as an R1 source
  they are a ceiling, not a dataset. The NYC TLC coordinate era (2009–2016) is the
  one thing that could change this and its schema is still unverified.
* **NGSIM** — CC BY-SA 3.0, the only freely redistributable micro dataset, 0.1 s,
  with `Section_ID` / `Int_ID` on the arterial corridors, so an edge list falls out
  without map matching. It would work. pNEUMA covers the same role with real
  lat/lon, a denser network and 500 k trajectories, so it is the better first
  choice; NGSIM is the fallback if the NC licence on pNEUMA is a problem.
* **LargeST / PeMS / METR-LA / PEMS-BAY / UTD19** — detector archives. Any TTE
  label is synthesised by our own composition rule, so we would be scoring a model
  against our own assumption; freeway-only except UTD19; no end-to-end ground
  truth exists in them at all. Legitimate as an R4 experiment, not as a city.
* **Helsinki Travel Time Matrix, Valhalla / r5r** — routing-engine output. Useful
  as a pretraining corpus or a topology probe; calling it TTE would be calling
  engine distillation TTE.
* **Shenzhen SciDB** (licence unstated, and the deposit is a 3.1 MB extract, not
  the 8.6 M trajectories), **IEEE 10.21227/7e1g-hw96** (thin metadata) — ten
  minutes each to check, not worth planning around.
* **Minneapolis scooters** — CC0 and real street-segment ids, but a few hundred
  thousand records a year and the vehicle is a scooter. Chicago and Austin are
  traps: their start and end centroids are byte-identical in the raw records.
* **DiDi GAIA / Grab-Posisi / Quebec-full** — letters, not code. Grab-Posisi is
  84 000 trajectories at 1 Hz plus a fifth geography; eVED now gives us 1 Hz
  without the wait, which lowers the urgency but not the value.
* **Dead ends**: NPMRDS (structurally unavailable to a non-US group), Uber Movement
  (service dead, the one claimed mirror has no manifest, no licence, zero
  downloads), Aalborg and Q-Traffic (no public release), Astana synthetic
  benchmark (simulated speeds, not measurements), MobilityBench (an LLM
  route-planning benchmark despite the name), HuggingFace (no ground-transport
  trip-level TTE dataset exists there at all).
* **Transit / maritime / aviation** (Dutch bus, Astana AVL, Swiss IstDaten, Warsaw
  ZTM, Delhi AVL, NOAA AIS, MARIS-Forecast, OpenSky) — real ETA labels for a
  different problem. Astana is the only CIS analogue of our setup and is CC BY 4.0
  on Zenodo with 1–5 s raw GPS, so it is the one to reach for if the transit task
  is ever taken up.
* **Yandex `urban-traffic-benchmark`** — MIT, metropolis-scale, but traffic-state
  forecasting, not TTE. Fits R4, not the table.

### The appendix's own item #1 still stands

`gctte.online` is NXDOMAIN and there is no mirror of Abakan/Omsk anywhere. Whatever
we add, the **re-release of Abakan/Omsk with a DOI** remains the item with the
highest ratio of citations to effort, and the only dataset only we can produce.

---

## 4. Suggested order

1. **eVED** — notebook runs, data verified in hand, 1 Hz, energy channels. Best
   value per day of work on this list.
2. **Quebec** — no download friction, already matched.
3. **San Francisco**, then **Rome** — one IEEE login each; SF buys a North American
   city, Rome buys the RED (PVLDB'25) comparison.
4. **GeoLife** — the multimodal angle, and a second Beijing to disambiguate the
   first one.
5. Send the **Grab-Posisi**, **DiDi GAIA** and **Quebec-full** letters in one
   sitting — an hour of work, and the waiting overlaps everything above.
6. **SUMO / LuST** counterfactuals — the only route to interventional TTE, and now
   a `pip install` away.
7. **pNEUMA** — as the noise-floor and composition-rule calibration section, not as
   a city.
8. **Citi Bike** — when the calibration / OD line of work actually starts.
9. Settle the **Beijing / T-Drive** question before the table is published either
   way.

## 5. Notes on running the notebooks

Every notebook writes the three gold files and ends with a `validate_gold` cell
that re-reads them and checks the contract (column names, equal list lengths,
`literal_eval` parses, `Total_time` consistency, and that every segment referenced
by a trip exists in the edge list and the geojson).

Verified by running:

| notebook | how it was tested |
|---|---|
| `quebec` | **real data**, produced 4 898 trips over 13 233 links |
| `eved` | **real data** (2 weekly files, 413 889 fixes, 1 024 trips); synthetic network stands in for the blocked Overpass |
| `sumo` | **real LuST network + a real SUMO run**, 270 trips converted |
| `citibike` | **real data** (a real month, 108 766 rides); synthetic network stands in for Overpass |
| `rome`, `san_francisco`, `geolife`, `pneuma` | synthetic traces in the exact raw formats + synthetic network — parsing, segmentation, filtering, matching, writing and the contract check all pass |

The four raw archives that need a manual download (IEEE DataPort ×2, Microsoft
GeoLife, EPFL pNEUMA) are all on hosts blocked from this session, so those
notebooks must be run where the network allows it.

Map matching, where it happens, is a small HMM: emission is snap distance,
transitions are restricted to same edge / 1-hop / 2-hop graph adjacency. The
adjacency term is what makes the direction of travel identifiable on two-way
streets — plain nearest-edge snapping picks the reverse edge roughly half the
time (measured: 0.29 connectivity vs 1.0 for the HMM on a test grid). Each
notebook reports per-trip connectivity and drops trips below `MIN_CONNECTIVITY`.
Spot-check a handful of matched routes on a map before training on any of it.
