# PCT Washington Interactive Planner

An interactive trip planning tool for the Washington section of the Pacific Crest Trail — Bridge of Gods (mile 2147) to the Northern Terminus at the Canadian border (mile 2653), a distance of 505.8 miles.

The planner uses dynamic programming to generate an optimal multi-night itinerary from your target pace, start date, and snowmelt conditions, scoring candidate campsites on elevation and date-aware mosquito pressure. You can adjust any night by dragging markers on the map, edit the campsite dataset directly in the browser, and export your plan as CSV or JSON.

**Live site:** https://dwettergreen.github.io/pct_planner/

---

## Project Structure

```
pct_planner/
├── index.html          ← Planner + embedded campsite editor (single page)
├── data/
│   ├── campsites.json  ← 171 campsites + 4 resupply stops
│   ├── trail.geojson   ← PCT Washington trail polyline (~9,000 GPS points)
│   └── plan.json       ← Saved plan; auto-restored on load if present
└── README.md
```

The tool is a static site — no server-side code, no database, no build step. It loads data from the `data/` folder via `fetch()`, which means it must be served over HTTP rather than opened directly from the filesystem. GitHub Pages handles this automatically; for local use run `python3 -m http.server 8000` in the project folder and open `http://localhost:8000`.

---

## Campsite Selection Algorithm

The planner uses **dynamic programming (DP)** to find the globally optimal sequence of campsites for the entire trip in one pass, rather than making greedy day-by-day decisions.

### Step 1 — Set the target night count

Given your average pace (say 13 mi/day) and the total distance (505.8 mi):

```
T = round(505.8 / 13) = 39 nights
```

### Step 2 — Define the daily distance window

To allow flexibility around the average, it uses a tolerance that starts tight and widens if no valid path is found:

```
min_d = 505.8 / (39 × 1.25) = 10.4 mi
max_d = 505.8 / (39 × 0.75) = 17.3 mi
```

If no complete path exists within those bounds (campsites are too sparse in a section), it retries with progressively wider tolerances: ±25%, ±35%, ±45%, ±55%, ±65%, ±80%. The flex slider controls the display of the range but the DP always finds a valid plan.

### Step 3 — Score each campsite

Every candidate campsite is scored on **elevation** and **mosquito pressure** on the estimated arrival date:

```
score = elev × (1 + 2.0 × (1 - mosquito_pressure))
```

Mosquito pressure is a value from 0.0 to 1.0, modeled as a bell curve (Gaussian) that peaks at a date determined by elevation band and your snowmelt date setting:

| Band | Elevation | Peak date (baseline Jun 1 melt) |
|------|-----------|----------------------------------|
| Valley | < 3,000 ft | June 4 |
| Montane | 3,000–4,500 ft | June 24 |
| Subalpine | 4,500–5,500 ft | July 11 |
| Alpine | > 5,500 ft | July 26 |

A camp at 5,000 ft arriving on July 11 gets peak mosquito pressure (1.0), collapsing the score multiplier to `1 + 2.0 × 0 = 1.0` — just raw elevation. The same camp in late August gets near-zero pressure, giving a multiplier of `1 + 2.0 × 1.0 = 3.0` — tripling the effective score.

### Step 4 — Dynamic programming

The DP fills a table where `dp[i]` = the best cumulative score achievable by stopping at campsite `i`. For each campsite it looks back at all earlier campsites within the valid distance window:

```
for each campsite i:
  for each earlier campsite j where min_d ≤ dist[i] - dist[j] ≤ max_d:
    candidate = dp[j] + score(i)
    if candidate > dp[i]: dp[i] = candidate, prev[i] = j
```

It then traces back from the terminus to reconstruct the winning sequence — the globally optimal path rather than locally good day-by-day choices.

Inter-campsite distances are derived from trail GPS geometry (the `trailDist` field in `campsites.json`), computed from the original Halfmile high-resolution track. The Halfmile mile marker is retained as a display reference only.

### What the algorithm does not consider

- **Terrain difficulty** — a strenuous pass day and a flat valley day with the same mileage score identically
- **Water carries** — the dataset notes water presence but the optimizer does not penalize dry stretches between camps
- **Permit requirements** — some areas (Enchantments, Glacier Peak Wilderness) have permit quotas the planner is unaware of
- **Trail closures** — fire or other seasonal closures are not reflected in the data
- **Camp crowding** — popular sites are not penalized for overuse

---

## Using the Planner

### Planning Controls

| Control | Effect |
|---------|--------|
| **Start date** | Date you leave Bridge of Gods; drives arrival date calculations and the mosquito model |
| **Avg mi/day** | Target pace; determines night count and DP distance window; rebuilds instantly on change |
| **Flex ± mi** | Displayed pace variation (informational; DP widens its window independently as needed) |
| **Snowmelt date** | Shifts mosquito pressure peaks for all elevation bands; push later for a heavy snow year |

### Reading the Itinerary

The sidebar shows a card per night with campsite name, PCT mile, elevation, miles that day, and arrival date. Badges flag notable conditions:

| Badge | Meaning |
|-------|---------|
| 🦟 HIGH / MEDIUM | Mosquito pressure on arrival date (low/minimal suppressed) |
| 🌬️ High Wind | Camp above 6,500 ft, likely exposed |
| 💨 Exposed | Camp 5,800–6,500 ft |
| 💧 Water | Water source at or near camp |
| 🚽 Outhouse | Outhouse present |
| Established | Designated site with infrastructure (pads, fire ring, bear box) |
| Resupply | One of four resupply stops; can serve as overnight camp |

Click any card to fly the map to that camp and open a details popup.

### Adjusting the Plan

Drag any numbered marker to a different campsite — the route line updates live and the itinerary rebuilds on release. Markers snap only to known campsites in `campsites.json`; they cannot be placed at arbitrary locations. To reset a night to the algorithm's choice, nudge the pace slider and return it — this triggers a full rebuild.

### Map

The **basemap selector** (top-right corner) switches between OpenTopoMap (recommended — PCT route visible), USGS Topo, and OpenStreetMap. The **elevation chart** at the bottom of the screen shows the full Washington section profile; click any bar to fly to that point.

### Exporting

The **Export** tab offers:

- **⬇ Download CSV** — one row per night; columns: PCT Mile, Miles Today, Campsite, Elevation, Date, Notes (type / water / mosquito risk / wind). Two blank columns are included for personal annotations. Opens in Excel or Google Sheets.
- **💾 Save plan.json** — compact JSON recording pace settings and any dragged overrides. Drop it in `data/` and commit to restore your plan automatically on next load.

---

## Editing the Campsite Data

### Editor Tab

Click the **Editor** tab. The pace controls are replaced by a campsite list, the map cursor becomes a crosshair, and all campsites appear as colored dots:

| Dot color | Source |
|-----------|--------|
| Orange | Halfmile GPS data (167 sites) |
| Green | Custom sites you have added |
| Blue square | Resupply stops |

**Add a campsite** — click anywhere on or near the trail. The location snaps to the nearest point in the trail GPS track, and a form opens pre-filled with that lat/lon and an estimated mile marker. Enter the name, elevation, type, and amenities and click **💾 Save**.

**Edit a campsite** — click a dot on the map or a row in the list. Edit the form and click **💾 Save**. Halfmile and custom sites can both be edited freely.

**Delete a campsite** — select it and click **🗑 Delete**. Resupply stops cannot be deleted.

**Search** — the search box filters by name or mile number.

### ⚠ Saving Changes Permanently

Edits exist only in the current browser session. Before closing the browser:

1. Click **⬇ Download campsites.json** in the Editor toolbar
2. Replace `data/campsites.json` in your local repo folder with the downloaded file
3. In GitHub Desktop: commit with a message like "Add camp at Lyman Lakes" → **Push origin**

The live site updates within a minute of pushing.

---

## Campsite Data Reference

| Field | Type | Description |
|-------|------|-------------|
| mile | number | NOBO PCT mile marker (Halfmile, display only) |
| trailDist | number | Distance from Bridge of Gods along trail geometry (used by optimizer) |
| name | string | Campsite name (no spaces preferred) |
| lat | number | Latitude (decimal degrees) |
| lon | number | Longitude (decimal degrees, negative W) |
| elev | number | Elevation in feet |
| type | string | `Undeveloped`, `Established`, or `Resupply` |
| water | boolean | Water source nearby |
| outhouse | boolean | Outhouse present |
| source | string | `halfmile`, `resupply`, or `custom` |
| desc | string | Notes / description |
| amenities | array | e.g. `["Lodging","Store","Laundry"]` |

---

## Known Limitations

- **Permit areas** — the Enchantments and parts of Glacier Peak Wilderness require permits; the planner does not know about quotas or reservation windows
- **Trail reroutes** — Washington gets several reroutes per year (fire damage, erosion). The campsite dataset reflects the trail as of the Halfmile source data; new campsites created by reroutes will not appear until added manually
- **Mosquito model** — the pressure curves are a simplified approximation based on elevation band and snowmelt date. Actual conditions vary by year, local microclimate, and whether you're near standing water
- **Resupply logistics** — the planner notes resupply locations but does not model mail drop deadlines, business hours, or package acceptance policies

---

## Data Sources

- **Campsites:** [Halfmile PCT Maps](https://www.pctmap.net/) GPS waypoint data
- **Trail geometry:** PCT Washington derived from Halfmile high-resolution GPS track
- **Resupply info:** [PCTA Resupply Guide](https://www.pcta.org/discover-the-trail/resupply/)

---

---

*Vibe coded with Claude by David Wettergreen*
