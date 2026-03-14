# PCT Washington Interactive Planner

An interactive trip planning tool for the Washington section of the Pacific Crest Trail — Bridge of Gods (mile 2147) to the Northern Terminus at the Canadian border (mile 2653), a distance of 505.8 miles.

The planner automatically generates an optimal multi-night itinerary based on your target pace, start date, and snowmelt conditions. It scores candidate campsites using elevation and date-aware mosquito pressure, selecting the sequence of stops that maximizes comfort across the entire trip. You can then adjust individual nights by dragging markers on the map, and export the result as a CSV or JSON file.

**Requirements:** A web server to serve the data files (GitHub Pages, or `python3 -m http.server` locally). A modern browser. No installation, no account, no dependencies beyond the files in this repo.

---

## Project Structure

```
pct_planner/
├── index.html          ← Interactive trip planner (includes embedded campsite editor)
├── data/
│   ├── campsites.json  ← 171 campsites + resupply stops (edit this to add sites)
│   ├── trail.geojson   ← PCT Washington trail polyline (9,359 GPS points)
│   └── plan.json       ← Last saved plan; restored automatically on load
└── README.md
```

---

## Running Locally

The planner loads data files via `fetch()`, which browsers block from `file://` URLs. You need a simple local web server:

```bash
cd pct_planner
python3 -m http.server 8000
```

Then open **http://localhost:8000** in your browser.

## Deploying to GitHub Pages

1. Push this folder to a GitHub repo
2. Go to repo **Settings → Pages → Source: Deploy from branch → main / (root)**
3. Your planner will be live at `https://USERNAME.github.io/pct_planner/`

Any time you update `campsites.json` and push, the live site reflects it immediately.

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

If no complete path exists within those bounds (campsites are too sparse in a section), it retries with progressively wider tolerances: ±25%, ±35%, ±45%, ±55%, ±65%, ±80%. The flex slider controls the display of the range but the DP uses these internal tolerances to guarantee it always finds a valid plan.

### Step 3 — Score each campsite

Every candidate campsite gets a score based on **elevation** and **mosquito pressure** on the estimated arrival date:

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

A camp at 5,000 ft arrived at on July 11 gets peak mosquito pressure (1.0), collapsing the score multiplier to `1 + 2.0 × 0 = 1.0` — just raw elevation. The same camp arrived at in late August gets near-zero pressure, giving a multiplier of `1 + 2.0 × 1.0 = 3.0` — tripling the effective score. The algorithm strongly prefers high camps but will trade elevation for a better mosquito window.

### Step 4 — Dynamic programming

The DP fills a table where `dp[i]` = the best cumulative score achievable by stopping at campsite `i`. For each campsite it looks back at all earlier campsites within the valid distance window:

```
for each campsite i:
  for each earlier campsite j where min_d ≤ (mile[i] - mile[j]) ≤ max_d:
    candidate = dp[j] + score(i)
    if candidate > dp[i]: dp[i] = candidate, prev[i] = j
```

It then traces back from the terminus to reconstruct the winning sequence. This guarantees the **globally optimal** path — the highest total score across all nights — rather than locally good day-by-day choices.

### What the algorithm does not consider

- **Terrain difficulty** — elevation gain/loss beyond what raw camp elevation implies; a strenuous pass day and a flat valley day with the same mileage score identically
- **Water availability** — all campsites in the dataset have water noted; dry camps are not penalized
- **Camp crowding** — popular sites are not penalized
- **Manual overrides** — once you drag a marker to a different camp, that night is locked and the DP result is overridden for that night only

---

## Using the Planner

### Controls

**Start date** — the date you leave Bridge of Gods (mile 2147). Used to compute arrival dates at each camp, which drive the mosquito pressure model.

**Snowmelt date** — the date when snow has melted enough to open high passes. Shifts the mosquito pressure peaks for all elevation bands; a late snow year pushes the peaks later.

**Avg mi/day** — your target daily mileage. Drives the night count and the DP distance window. Adjusting this slider immediately rebuilds the optimal plan.

**Flex ± mi** — the displayed variation around your average (e.g. ±3 means shown range is 10–16 mi/day at avg 13). This is informational; the DP internally widens its search window as needed.

### Reading the Itinerary

The left sidebar shows a card for each night with the campsite name, PCT mile, elevation, miles for that day, and the arrival date. Badges flag notable conditions:

- **🦟 HIGH / MEDIUM** — mosquito pressure on your arrival date (low and minimal are suppressed)
- **🌬️ High Wind** — camp above 6,500 ft, likely exposed
- **💨 Exposed** — camp between 5,800–6,500 ft
- **💧 Water** — water source at or near the camp
- **🚽 Outhouse** — outhouse present
- **Established** — designated site with infrastructure (tent pads, fire ring, bear box)
- **Resupply** — one of the four resupply points; can be used as an overnight stop

Click any sidebar card to fly the map to that camp and open its popup with full details.

### Moving Campsites

The numbered markers on the map correspond to each night. You can drag any marker to a different campsite:

1. Click and hold a numbered marker — it becomes draggable
2. Drag it toward your preferred campsite; the route line updates live as you drag
3. Release — the marker snaps to the nearest campsite in the dataset and the itinerary rebuilds instantly

The marker will only snap to known campsites in `campsites.json` — it cannot be dropped on an arbitrary map location. If a campsite you want is not in the dataset, add it first using the editor (see below). To reset a night back to the algorithm's choice, adjust the pace slider slightly and back — this triggers a full rebuild, discarding all manual overrides.

### Map Controls

The **basemap selector** (top-right) offers three options:
- **OpenTopoMap** — recommended; shows PCT route and terrain clearly
- **USGS Topo** — detailed US government topographic map
- **OpenStreetMap** — road and place names, less terrain detail

The **legend** (bottom-left) shows the elevation color scale used for campsite markers. The **elevation chart** at the bottom shows the full Washington section profile; click any bar to fly the map to that point on the trail.

### Exporting Your Plan

The **Export** tab in the sidebar offers two options:

**⬇ Download CSV** — a spreadsheet with one row per night. Columns: PCT Mile, (blank), (blank), Miles Today, (blank), Campsite, Elevation, Date, Notes. The Notes column contains a comma-separated string of campsite type, water, mosquito risk, and wind exposure. The blank columns are provided for your own annotations. Opens directly in Excel or Google Sheets.

**💾 Save plan.json** — downloads a compact JSON file recording your pace settings and any campsites you manually moved. To restore the plan automatically on your next visit, replace `data/plan.json` in the repo with this file and commit it.

---

## Adding and Editing Campsites

### Using the Editor Tab

Click the **Editor** tab in the sidebar. The planner switches into editing mode — the pace controls are hidden, the map cursor becomes a crosshair, and all campsites appear as colored dots:

- **Orange dots** — Halfmile source data (167 sites)
- **Green dots** — custom sites you have added
- **Blue squares** — resupply stops

**Adding a new campsite** — click anywhere on or near the trail. The marker snaps to the nearest GPS point in the trail data, and a form opens pre-filled with that lat/lon and an estimated mile marker. Fill in the name, elevation, type, and amenities, then click **💾 Save**.

**Editing an existing campsite** — click any dot on the map, or find it in the scrollable list and click the row. The form opens with current values. Make your changes and click **💾 Save**. Both Halfmile and custom sites can be edited; resupply stops cannot be deleted.

**Deleting a campsite** — select it and click **🗑 Delete**. You will be asked to confirm. The deletion is immediate in the current session.

**Searching** — the search box filters the list by name or mile number.

### ⚠ Making Changes Permanent

Edits in the Editor tab are live for the current browser session only. To save them permanently:

1. Click **⬇ Download campsites.json** in the Editor toolbar — do this before closing the browser or your edits will be lost
2. Copy the downloaded file into your local `pct_planner/data/` folder, replacing the existing file
3. In GitHub Desktop, write a commit message (e.g. "Add camp at Lyman Lakes") and click **Commit to main**
4. Click **Push origin** — the live site updates within a minute
