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

- **🔗 Generate Link** — encodes your settings (pace, dates, dragged campsites) into a URL you can bookmark or share
- **⬇ Download CSV** — downloads a spreadsheet of all nights with dates, miles, elevation, and risk indicators
- **📋 Copy to Clipboard** — plain text itinerary to paste anywhere

To save a plan to `plan.json` so it loads automatically on the next visit, download the CSV/JSON from the Export tab and commit `data/plan.json` to your repo.

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
