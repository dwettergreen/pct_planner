# PCT Washington Interactive Planner

A self-hosted trail planning tool for the PCT Washington section (Bridge of Gods → Northern Terminus, miles 2147–2653).

## Project Structure

```
pct_planner/
├── index.html          ← Interactive trip planner
├── editor.html         ← Campsite data editor
├── data/
│   ├── campsites.json  ← All 171 campsites + resupply stops (edit this!)
│   ├── trail.geojson   ← PCT Washington trail polyline (9,359 GPS points)
│   └── plan.json       ← Last saved plan (updated by Export tab)
└── README.md
```

## Running Locally

Because the planner loads data files via `fetch()`, you need a local web server — browsers block `fetch()` from `file://` URLs.

```bash
cd pct_planner
python3 -m http.server 8000
```

Then open **http://localhost:8000** in your browser.

## Deploying to GitHub Pages

1. Push this folder to a GitHub repo (e.g. `pct_planner`)
2. Go to repo **Settings → Pages → Source: Deploy from branch → main / (root)**
3. Your planner will be live at `https://USERNAME.github.io/pct_planner/`

Any time you update `campsites.json` and push, the live site reflects it immediately.

## Adding Campsites

### Option A — Campsite Editor (recommended)
1. Open `http://localhost:8000/editor.html`
2. Click **＋ New Campsite**, fill in the form
3. Use **📍 Pick on Map** to click the exact location
4. Click **Save**, then **⬇ Save JSON**
5. Replace `data/campsites.json` with the downloaded file
6. Commit and push to update the live site

### Option B — Edit JSON directly
Open `data/campsites.json` in any text editor. Each campsite is one object:

```json
{
  "mile": 2312.5,
  "name": "MyNewCamp",
  "lat": 46.789,
  "lon": -121.234,
  "elev": 4800,
  "type": "Undeveloped",
  "water": true,
  "outhouse": false,
  "source": "custom",
  "desc": "Nice flat spot near creek, good bear hang trees"
}
```

**Required fields:** `mile`, `name`, `lat`, `lon`  
**Optional fields:** `elev`, `type`, `water`, `outhouse`, `desc`, `amenities`  
**Source values:** `halfmile` (original data), `resupply`, `custom` (your additions)

## Saving Your Plan

In the planner, the **Export** tab offers:

- **⬇ Download CSV** — downloads a spreadsheet of all nights with PCT mile, miles today, campsite, elevation, date, and notes (type / water / mosquito risk / wind exposure)
- **💾 Save plan.json** — downloads a `plan.json` file; place it in the `data/` folder and commit to restore your plan automatically on next load. Saves pace settings and any manually dragged campsites.

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

If no complete path exists within those bounds (campsites are too sparse in a section), it retries with progressively wider tolerances: ±25%, ±35%, ±45%, ±55%, ±65%, ±80%. Your flex slider controls the display of the range but the DP uses these internal tolerances to guarantee it always finds a valid plan.

### Step 3 — Score each campsite

Every candidate campsite gets a score based on **elevation** and **mosquito pressure** on your estimated arrival date:

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

A camp at 5,000 ft arrived at on July 11 gets peak mosquito pressure (1.0), collapsing the score multiplier to `1 + 2.0 × 0 = 1.0` — just raw elevation. The same camp arrived at in late August gets near-zero pressure, giving a multiplier up to `1 + 2.0 × 1.0 = 3.0` — tripling the effective score. In practice the algorithm strongly prefers high camps, but will trade elevation for a better mosquito window.

### Step 4 — Dynamic programming

The DP fills a table where `dp[i]` = the best cumulative score achievable by stopping at campsite `i`. For each campsite it looks back at all earlier campsites within the valid distance window:

```
for each campsite i:
  for each earlier campsite j where min_d ≤ (mile[i] - mile[j]) ≤ max_d:
    candidate = dp[j] + score(i)
    if candidate > dp[i]: dp[i] = candidate, prev[i] = j
```

It then traces back from the terminus to reconstruct the winning sequence. This guarantees the **globally optimal** path — the highest total score across all nights — rather than locally good day-by-day choices.

### How daily distances are calculated

Daily distance is the difference between consecutive NOBO mile markers from the Halfmile data:

```
miles_today = camp[night_i].mile - camp[night_i-1].mile
```

The Halfmile mile marker is cumulative distance along the PCT trail centerline from the southern terminus, so differences do reflect trail distance (accounting for switchbacks and curves) rather than straight-line distance. However the algorithm treats all miles as equal effort — a 14-mile day over a 7,000 ft pass scores the same as a 14-mile flat valley day. For particularly strenuous segments (Glacier Peak, Harts Pass approach) you may want to manually drag those nights to shorter camps.

### What the algorithm does not consider

- **Water** — all campsites in the dataset have water noted, but dry camps are not penalized
- **Terrain difficulty** — elevation gain/loss beyond what raw elevation implies
- **Camp crowding** — popular sites are not penalized
- **Manual overrides** — once you drag a marker to a different camp, that night is locked and the DP result is overridden for that night only

## Campsite Fields Reference

| Field      | Type    | Description                              |
|------------|---------|------------------------------------------|
| mile       | number  | NOBO PCT mile marker                     |
| name       | string  | Campsite name (no spaces preferred)      |
| lat        | number  | Latitude (decimal degrees)               |
| lon        | number  | Longitude (decimal degrees, negative W)  |
| elev       | number  | Elevation in feet                        |
| type       | string  | `Undeveloped`, `Established`, `Resupply` |
| water      | boolean | Water source nearby                      |
| outhouse   | boolean | Outhouse present                         |
| source     | string  | `halfmile`, `resupply`, `custom`         |
| desc       | string  | Notes / description                      |
| amenities  | array   | e.g. `["Lodging","Store","Laundry"]`     |

## Data Sources

- **Campsites:** [Halfmile PCT Maps](https://www.pctmap.net/) GPS waypoint data
- **Trail:** PCT Washington GeoJSON (simplified from Halfmile source, 9,359 points)
- **Resupply info:** [PCTA Resupply Guide](https://www.pcta.org/discover-the-trail/resupply/)
