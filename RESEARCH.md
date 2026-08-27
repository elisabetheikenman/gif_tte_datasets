# New TTE datasets — what we can use, and what it costs

Scope: the three sources you sent, plus a pass over `TTE_datasets_appendix.md` so
nothing in it is silently dropped. Checked 27 Aug 2026.

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

So a source is usable for **R1 (route-aware)** only if it gives us, per trip, a
sequence of road segments with timestamps — either already matched, or raw GPS
dense enough that we can map-match it ourselves. Everything that only gives
origin, destination and a duration is **OD-only**: still usable, but for a
different task, and not a drop-in for the current table.

---

## 1. The three sources you sent

### 1.1 Quebec City — `github.com/melmasri/traveltimeCLT` ✅ USE

**Verified by downloading and reading the data.**

- `data/trips.rda` in the repo (7.3 MB), identical `tripset` in `melmasri/traveltimeHMM`.
- **4 914 trips / 322 799 link traversals / 13 235 links**, 28 Apr – 16 May 2014.
- Columns: `tripID`, `linkID`, `timeBin`, `speed`, `duration_secs`,
  `distance_meters`, `entry_time`. Source: anonymised *Mon Trajet* smartphone GPS
  (Brisk Synergies); package licence **GPL-3**, no separate data licence.
- **Already map-matched.** Each row is one link of one trip with entry time and
  traversal duration; `entry_time[k+1] ≈ entry_time[k] + duration[k]`
  (corr(trip span, sum of link durations) = **0.991**), so the links really are
  contiguous. This is the R1 signal with zero matching work on our side.
- **What is missing: geometry.** `linkID` is an anonymised integer — no
  coordinates, no OSM id, no shapefile. The network was never released, so
  `road_network_unique_osmids_quebec.geojson` **cannot be produced**, and anything
  in our pipeline that consumes coordinates (map crops, spatial embeddings,
  position-based GNN features) cannot run on Quebec.
- Also: the sample is almost entirely weekday rush hour (114 `Weekendday` and 112
  `EveningNight` rows out of 322 799), and the paper used 19 967 trips against the
  4 914 released.
- **Action:** none beyond running the notebook — `git clone` and go. Write to
  Elmasri (`elmasri.m@use.startmail.com`) if we want the full 19 967 trips and/or
  the link geometry; that letter is cheap and the upside is a fully usable city.
- **Notebook:** `prepare_dataset_quebec.ipynb` (runs, verified).

### 1.2 Rome taxi — `ieee-dataport.org/open-access/crawdad-romataxi` ✅ USE, with work

**The IEEE DataPort page is blocked by this session's egress proxy, so the record
itself was not re-read here.** Format and size confirmed from the CRAWDAD
converter source (`github.com/julianofischer/roma-taxi-converter`) and secondary
sources; download requires a free IEEE account and has to be done by hand.

- ~320 taxis, **~21.8 M fixes**, 1 Feb – 2 Mar 2014, ~7 s nominal sampling,
  ~374 MB (1.5 GB raw). No licence stated on the record — cite Amici et al.,
  *MoWNeT 2014*, and ship a loader, not a copy of the data.
- One file, `taxi_february.txt`, `;`-separated,
  `id;2014-02-01 00:00:00.739166+01;POINT(lat lon)` — **latitude first**.
- **What has to be built:** trip segmentation (there is no occupancy flag and no
  navigation session, so trips are cut on time gaps and standstills) and map
  matching against OSM. Both are in the notebook.
- **What we lose:** trip boundaries are *inferred*, so `Total_time` means "time
  driving between two stops", not "duration of a requested route". A cruising taxi
  is also not a private car. And OSM 2026 is not Rome 2014.
- **Why bother:** Rome is the city RED (PVLDB 2025) uses, so it buys direct
  comparability with a current VLDB baseline, plus an irregular-layout European city.
- **Notebook:** `prepare_dataset_rome.ipynb` (logic verified against a synthetic
  network; the OSM download and the real archive must be run on your machine).

### 1.3 San Francisco Cabspotting — `ieee-dataport.org/open-access/crawdad-epflmobility` ✅ USE, cheap

**Same caveat: the DataPort page is blocked here;** format confirmed from
secondary sources. Free IEEE account, DOI `10.15783/C7J010`, attribution licence.

- **536 cabs, ~11.2 M fixes**, 17 May – 10 Jun 2008, ~90 MB.
- `_cabs.txt` + one `new_<cab>.txt` per cab; each line is
  `latitude longitude occupancy epoch`, **newest fix first**.
- **The occupancy flag is the reason to prefer this over Rome**: a maximal run of
  `occupancy == 1` is a real hired trip with a real start and end, so trip
  boundaries are observed rather than guessed.
- **The catch is the sampling rate.** The appendix (and the DataPort record) says
  "< 10 s"; what the files actually contain is closer to **one fix per minute**.
  At 60 s a cab covers several hundred metres, so the matched edge sequence is an
  inference between fixes. The notebook prints the real interval quantiles so we
  can decide with numbers rather than with the record's claim.
- Plus: 2008-era GPS on a 2026 OSM network, downtown one-way grid.
- **Why bother:** first North American city in the table, and it is a day of work.
- **Notebook:** `prepare_dataset_san_francisco.ipynb` (logic verified against a
  synthetic network).

---

## 2. The rest of the appendix — verdicts

Statuses below are the appendix's own (2 Aug 2026) unless marked *(checked here)*.
I did not re-verify each link: several relevant hosts (`ieee-dataport.org`,
`huggingface.co`, `overpass-api.de`, `impactcybertrust.org`) are blocked by this
session's egress policy.

### Route-aware, worth adding after the three above

| Dataset | Verdict | What it costs |
|---|---|---|
| **GeoLife 1.3** (Microsoft) | **Use.** Direct download, 17 621 trajectories, 1–5 s sampling, 2007–2012, **transport-mode labels** — the only open source for cross-mode TTE transfer. | Same pipeline as Rome (segment + match), Beijing network. ~3 days. |
| **T-Drive** (Beijing, 10 357 taxis) | **Check first.** The appendix argues our existing `Beijing` column probably *is* T-Drive. If so, adding it is double-counting; if not, our Beijing row needs a source. | Half a day of checking, then either drop or document. |
| **Porto** `kraina/porto_taxi` (HF, CC BY 4.0) | **Use as a reproducibility anchor**, ~400 k trajectories, 15 s polylines. Standardised version of UCI #339. | Trivial: polylines + 15 s cadence → same matcher. ~2 days. |
| **eVED** (Ann Arbor) | **Use if the mirror works.** Already Valhalla-matched, >99 % of records on-road, 22 M rows, real routes + energy. The GitHub in the paper is 404; only an unverified Bitbucket clone remains, and the licence is unstated. | 1–2 weeks, plus a licence question. |
| **Shenzhen** `10.57760/sciencedb.j00133.00519` | **Do not plan on it.** Licence unstated, and the deposit is a 3.1 MB curated extract, not the 8.6 M trajectories. | Ten minutes to check rights, then decide. |

### OD-only — a different task, not a drop-in

Chicago TNP + Taxi (**≈521 M + 218 M trips**, coordinates are census-tract
centroids, time rounded to 15 min), NYC TLC (zone ids since 2015; the 2009–2016
coordinate-era parquet files are delisted but still served), the four bikeshare
systems (Citi Bike / Divvy / Capital Bikeshare / Bay Wheels), Helsinki Travel Time
Matrix (a *routing-engine output*, not a measurement), Minneapolis scooters.
All are real measured durations with unobserved routes. They are the right
material for scaling / fairness / distribution-shift / calibration questions and
the wrong material for our current R1 table. **Note the redistribution ban** on
Divvy and Capital Bikeshare — loader and preprocessing script only, never a
repackaged benchmark file.

### Needs a letter, not code

**DiDi GAIA** (Xi'an, Chengdu — application form), **Grab-Posisi** (84 000
trajectories at **1 Hz**, the highest rate in any public set, Singapore + Jakarta —
email `grab.posisi@grabtaxi.com`), **Quebec full set** (see 1.1). Cheap to send,
weeks to wait; send them now if we want them at all.

### Not usable for this task

- **NPMRDS / RITIS** — structurally unavailable to a non-US academic group.
- **Uber Movement** — service dead (404); the one claimed raw mirror has no
  manifest, no licence and zero downloads.
- **Chicago / Austin scooters** — start and end centroids are byte-identical in
  the raw records; there is no OD signal to recover.
- **Aalborg** (MM-Path), **Q-Traffic**, canonical **Shenzhen / Hangzhou** taxi —
  no public release.
- **LargeST / PeMS / METR-LA / UTD19** — detector archives. Any TTE label is
  *synthesised by our own composition rule*, so we would be scoring a model against
  our own assumption. Freeway-only except UTD19. Fine as an R4 experiment, not as
  a new city.
- **LuST / MoST / InTAS / MATSim / CBLab** — simulation. Paired counterfactuals
  are their unique value; the labels are car-following outputs. Diagnostic bench,
  never a headline benchmark, never mixed into training without a domain flag.
- **pNEUMA / NGSIM** — ~1 km², hours. Calibration set for the composition rule,
  not a TTE benchmark.
- **Astana synthetic congestion benchmark** — speeds are simulated, not measured.
- **MobilityBench** — an LLM route-planning benchmark despite the name.
- **HuggingFace** — no ground-transport trip-level TTE dataset exists there.
- **Transit / maritime / aviation** (Dutch bus, Astana AVL, Swiss IstDaten, Warsaw
  ZTM, Delhi AVL, AIS, OpenSky) — real ETA labels, different problem. Only relevant
  if we take one of the transit tasks.

### The appendix's own item #1 still stands

`gctte.online` is NXDOMAIN and there is no mirror of Abakan/Omsk anywhere. Whatever
we do with new cities, the **re-release of Abakan/Omsk with a DOI** is the item
with the highest ratio of citations to effort, and it is the one dataset only we
can produce.

---

## 3. Suggested order

1. **Quebec** — done, notebook runs, no download friction.
2. **San Francisco** — one login, one day, first North American city.
3. **Rome** — one login, two days, buys the RED comparison.
4. Send the **Grab-Posisi**, **DiDi GAIA** and **Quebec-full** letters in the same
   sitting; they cost an hour and the waiting runs in parallel with everything else.
5. **GeoLife** next if we want the multimodal angle; **Porto** if we want a
   reproducibility anchor.
6. Decide the **Beijing / T-Drive** question before publishing the table either way.

## 4. Notes on running the notebooks

`prepare_dataset_rome.ipynb` and `prepare_dataset_san_francisco.ipynb` download the
road network from OSM via `osmnx` and cache it as GraphML. Both the IEEE archives
and the Overpass API are unreachable from the session this was written in, so those
two notebooks were validated end-to-end against a synthetic road network and
synthetic traces in the exact raw formats — parsing, segmentation, filtering,
matching, writing and the format check all pass. Run them against the real archives
on a machine with normal network access. `prepare_dataset_quebec.ipynb` was run on
the real data and produced 4 898 trips over 13 233 links.

Map matching is a small HMM (emission = snap distance, transition = graph
adjacency, restricted to same / 1-hop / 2-hop). The adjacency term is what makes
the direction of travel identifiable on two-way streets — plain nearest-edge
snapping picks the reverse edge roughly half the time. Each notebook reports
per-trip connectivity and drops trips below `MIN_CONNECTIVITY`; spot-check a
handful of matched routes on a map before training on any of it.
