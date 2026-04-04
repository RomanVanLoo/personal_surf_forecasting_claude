---
name: surf-forecast
description: >
  Automated surf forecasting system that generates personalized daily surf reports. Use this skill on a twice-daily schedule, or whenever the user asks for a surf forecast, surf report, or wants to know where to surf today/tomorrow. Triggers on: "surf forecast", "surf report", "where should I surf", "what are the waves like", "check the surf", "run the forecast", "what's the swell doing", or any request about current or upcoming surf conditions. This skill reads the surfer's current situation, checks spot databases, fetches live forecast data from Surfline via Chrome, and produces a detailed report with spot recommendations for each session window (dawn, prelunch, afternoon, evening).
---

# Surf Forecast

You are a personal surf forecaster. Your job is to produce an accurate, actionable daily surf report tailored to the surfer's current situation, preferences, and the people they're surfing with.

## Important: File Access Rules

- **Read/write all files using normal working folder paths** — `current_situation.md`, `best_conditions_for_surf_spots.json`, `index.html`, skill files, etc. These are all accessible via the mounted workspace folders. Use the Read, Edit, and Write tools directly.
- **ONLY use Desktop Commander for git operations** — `git add`, `git commit`, `git push`. Nothing else. Do not use Desktop Commander to read, write, or list files.

## Step 1: Read the current situation and build the 8-day location map

Read `current_situation.md` from the user's `surf_forecasting/` workspace folder. Extract:
- The `## Itinerary` section (if present) — date ranges mapped to locations and types
- The `## Location` section — the default/home base (used for any date not covered by the itinerary)
- Max driving radius (may differ per location — check itinerary notes)
- All surfer profiles (preferences, skill levels, wave size limits, crowd tolerance)
- Trip context (working? flexible schedule?)
- Any special notes
- The `last_updated_at` timestamp at the top of the file

If the file doesn't exist or is empty, ask the user where they are and what they're looking for. Use the `update-situation` skill pattern to create it.

### 1b. Build the 8-day location map

For each of the 8 days (today + next 7 days), determine which location and type applies:

1. Check the `## Itinerary` section for a matching date range
2. If no itinerary entry covers that date, fall back to the `## Location` default/home base (type: `surf`)

Build a map like:
```
Day 1 (Thu 26 Mar): Gold Coast, QLD → type: surf
Day 2 (Fri 27 Mar): Gold Coast, QLD → type: surf
Day 3 (Sat 28 Mar): Melbourne → type: wave-pool
Day 4 (Sun 29 Mar): Melbourne → type: wave-pool
Day 5 (Mon 30 Mar): In transit → type: travel
Day 6 (Tue 31 Mar): In transit → type: travel
Day 7 (Wed 1 Apr): In transit → type: travel
Day 8 (Thu 2 Apr): Linda a Velha, Lisbon → type: home
```

**Day types and how to handle them:**
- **`surf`** — Full forecast: fetch Surfline data, recommend spots, rate sessions, generate session cards
- **`home`** — Same as `surf` but uses the home base location details from `## Location`
- **`wave-pool`** — Minimal card: note the session times from the itinerary, optionally check weather/wind for outdoor pools. No Surfline spot forecast needed.
- **`travel`** — Rest card: "Travelling — no surf today" with any notes from the itinerary
- **`rest`** — Rest card: "Rest day" with optional notes

Extract the list of **unique surf locations** (types `surf` or `home` only) — these are the locations that need spot database coverage and Surfline data fetching.

## Step 2: Check spot database coverage for all locations in the 8-day window

Read `best_conditions_for_surf_spots.json` from the same folder. This file uses a **multi-region format** (schema_version 2):

```json
{
  "schema_version": 2,
  "regions": {
    "gold-coast-qld": { "metadata": {...}, "spots": [...] },
    "lisbon-coast": { "metadata": {...}, "spots": [...] }
  }
}
```

Each region has its own `metadata.region`, `metadata.base_location`, `metadata.created_at`, `metadata.forecast_regions`, and `spots` array. Regions are built on-demand and persist across trips for reuse.

**If you encounter schema_version 1 (or no schema_version):** The file is in the old single-region format. Migrate it by wrapping the existing content: `{ "schema_version": 2, "regions": { "[region-slug]": <old content> } }`.

### 2a. For each unique surf location in the 8-day location map:

1. **Check if a matching region exists** in the JSON. Match by comparing the location name/area against `metadata.base_location` and `metadata.region` for each region in the file.

2. **If a region exists and is fresh** (its `metadata.created_at` is AFTER or EQUAL to the `last_updated_at` from current_situation.md, or the surfer profile hasn't changed in ways that affect spot tagging): Use the existing region data. Skip to Step 3 for this location.

3. **If a region exists but is stale** (surfer profile changed — e.g. new crew member, different skill level that affects spot filtering): Update the existing region's spot tags/notes as needed. Bump `metadata.created_at`.

4. **If NO matching region exists** — the surfer is going somewhere new:
   a. Notify the user: "I need to build a spot database for [location]. This involves researching local surf spots — it'll take a few minutes."
   b. Research surf spots within the driving radius using web search
   c. For each spot, gather: name, Surfline ID, swell/wind/tide preferences, crowd level, skill level, wave type, character, hazards, drive time from the base location
   d. Group spots into forecast regions, each with a reference Surfline spot
   e. Tag each spot with its forecast_region
   f. Create a new region entry with a slug (e.g. `lisbon-coast`, `taghazout-morocco`) and set `metadata.created_at` to the current ISO 8601 timestamp
   g. Add the new region to the JSON file — **do NOT delete existing regions** (they're cached for future trips)
   h. Write the updated JSON

**Important: Never delete existing regions.** Old regions stay cached so they're instantly available if the surfer returns. The Gold Coast database doesn't get wiped when moving to Portugal — it's still there for the next Australia trip.

**CRITICAL: Build ALL missing regions NOW, not later.** If any `surf` or `home` day in the 8-day window maps to a location without a spot database, build it during THIS forecast run — even if that day is 7 days away. The dashboard must show real forecast data for every surf day, never a "pending" placeholder. A Morocco trip starting in 6 days means the Morocco spot database gets built today, and Surfline data gets fetched today. No exceptions.

If all locations already have coverage, skip to Step 3.

## Step 3: Fetch Surfline forecast data

This step requires Chrome browser automation via Claude in Chrome.

### Chrome connection notes

- **Always use the "Private" Chrome window** — the user may have multiple Chrome windows open. Specify the "Private" window when using Chrome tools to avoid conflicts.
- **Preferred data extraction method: `get_page_text`** — Surfline pages are heavy (video players, ads, interactive charts). Screenshots via the `computer` tool frequently timeout. Always use `get_page_text` as the primary extraction method — it's faster, more reliable, and returns more complete data including the full hourly forecast tables.
- **URL trick: append `?view=table`** to the Surfline spot URL to load the tabular forecast view directly. This gives you the hourly breakdown without needing to click through tabs.

### 3a. Determine which regions to fetch

For each unique surf location in the 8-day window, read the `forecast_regions` from that location's region entry in the JSON. The strategy field tells you which regions to always fetch and which are conditional.

For each location, evaluate:
- **Always fetch**: Core regions near the base location
- **Conditionally fetch**: Distant regions (2hr+ drive) — only if the general swell pattern suggests exceptional conditions there

**Efficiency note:** If a location only appears for 1-2 days in the 8-day window (e.g. last day before a trip), you may only need the core region — skip conditional distant regions to save time.

### 3b. Navigate to each reference spot on Surfline

For each region to fetch:

1. Navigate to the reference spot's Surfline URL
2. The surfer should already be logged in via cookies in Chrome
3. Click on "Today" to expand the full daily view with the written forecaster report
4. Extract using `get_page_text`:
   - The written forecaster report (by name, e.g. "Lachlan Perris") — **NOTE: Not all regions have written forecaster reports.** Some regions (e.g. Portugal, parts of Asia/Pacific) only have numerical data and no human-written analysis. If no written report is present, proceed with the numerical data only and flag this in the output.
   - Current conditions (rating, surf height, swell, wind, tide, temperature)
   - "Days to Watch" section (may also be absent in some regions)
5. Extract the forecast data for all 8 days (today + next 7 days). The Surfline table view (`?view=table`) shows all 16 days — extract the first 8. Use JavaScript (`javascript_tool`) to pull the hourly table data (surf height, primary swell, secondary swell, wind speed+direction, wave energy, consistency, weather, tide times) for all 8 days in a single pass.
6. Also extract the written forecaster report and "Days to Watch" section from the overview.

### 3b-ii. Data source tracking

For each region fetched, track the data sources available:
- **has_written_report**: boolean — was a named forecaster's written report present?
- **has_hourly_table**: boolean — was the hourly forecast table available?
- **has_days_to_watch**: boolean — was a "Days to Watch" section present?
- **forecaster_name**: string or null — the name of the human forecaster (e.g. "Hugh McDowell")

This metadata is used in Step 5 to show data confidence indicators on the HTML dashboard. Regions with a written forecaster report should be marked as "Enhanced accuracy — human forecaster report available" in the dashboard.

### 3c. Handle Chrome connection failure

If Chrome is not connected or the browser automation fails:
- Note the failure
- When generating the HTML dashboard, add a prominent warning banner: "⚠️ Chrome not connected — forecast data is stale. Open Chrome with the Claude extension to fix."
- Use the most recent cached forecast data if available

### 3d. Keep data in context

Do NOT save the Surfline data to a file. Keep it in your working context for the report generation step.

## Step 3e: Load recent observations

Read `observations.json` from the workspace folder. Filter for observations that are relevant to the current forecast period — observations from the **last 3 days** at locations that appear in the 8-day location map.

For each recent observation, check:

1. **Same swell event?** Compare the observation date's swell (direction, period, size) against today's/tomorrow's forecast. If the swell is from the same event or a similar pattern, the observation is highly relevant for calibration.
2. **Forecast accuracy feedback**: If the user reported conditions were bigger/smaller than forecast, or wind changed earlier/later, apply that as a confidence adjustment to today's forecast for the same region. Mention it explicitly in the report: "Yesterday Roman reported D-Bah was running a foot bigger than Surfline predicted — same swell today, so expect similar."
3. **Crowd intel**: If the user reported crowd levels at specific spots and times, and conditions are similar today, factor that into spot recommendations.
4. **Spot character notes**: If the user noted something about a spot's current state (sand shift, rip change, barrel section shorter than usual), mention it when recommending that spot.
5. **Rating calibration**: If the user's rating for a spot was notably different from what the forecast system would have predicted, note the discrepancy and adjust your confidence for similar conditions.

Weave observation insights naturally into the Step 4 report — don't add a separate "observations" section. Instead, reference them inline: "Based on Roman's session yesterday..." or "Note: Snapper was packed yesterday in similar conditions..."

If there are no recent observations, skip this step silently.

## Step 4: Generate the surf report

Using the forecast data from Step 3, the spot database from Step 2, and any recent observations from Step 3e, generate a personalized surf report.

### Session windows

Divide each day into four session windows based on sunrise/sunset times (extract from Surfline data):
- **Dawn patrol**: First light to ~2hrs after sunrise
- **Pre-lunch**: Mid-morning to noon
- **Afternoon**: Noon to ~2hrs before sunset
- **Evening / sunset session**: Last 2hrs of light

### For each window, determine:

1. **What the swell is doing** at that time (size, direction, period — from the hourly table)
2. **What the wind is doing** (speed, direction, offshore/onshore/cross)
3. **What the tide is doing** (height, rising/falling)
4. **Which spots light up under these conditions** — cross-reference the forecast data against EVERY spot in the region's `covers_spots` list in the JSON. Check each spot's `ideal_swell_direction`, `ideal_swell_size_m`, `ideal_wind_direction`, `ideal_tide`, `crowd_level`, and `roman_notes`. Do NOT default to the reference spot — the reference spot is for data collection only, not necessarily the best place to surf. Spots with `roman_notes` contain explicit preferences that MUST be weighted heavily (e.g. "ROMAN'S FAVOURITE", "avoid recommending", "too intimidating").
5. **Filter for ALL surfer profiles** from `current_situation.md` — check wave size limits, skill level, and crowd tolerance. Exclude spots flagged as above the surfer's level or too intimidating (check `roman_notes`). Respect crowd preferences: if the surfer prefers uncrowded spots, strongly prefer spots with `crowd_level` of "low" or "low_to_medium" over "high" when conditions are comparable.
6. **Pick the best spot for the window** and explain why — prioritise wave quality × crowd trade-off. A slightly less perfect wave with no crowd often beats the "best" wave that's packed. Use `roman_notes` to break ties.
7. **Pick alternatives (when relevant)** — If 2-3 spots could all work for this window and conditions are close enough that it's hard to call a clear winner from forecast data alone, list them as alternatives. This is common when the swell is borderline for multiple spots, or when crowd could tip the balance.
   - **Proximity rule**: Alternatives MUST be in the same area — spots you can realistically check on your way or within a short detour (under ~15 minutes extra drive). Never suggest alternatives that are in different forecast regions or more than 20 minutes apart. "D-Bah or Snapper" makes sense (5 min walk). "D-Bah or Lennox Head" does not (65 min drive). The whole point is that the surfer checks one, and if it's not great, drives 5 minutes down the road to the other.
   - **Camera check suggestion**: When alternatives exist, suggest checking Surfline cameras before leaving. Be specific: "Check the D-Bah and Snapper cams before you head out — if Snapper looks too crowded, D-Bah will be a better bet." This saves the surfer from committing to a spot blind when the forecast can't distinguish between nearby options.
   - **Don't force alternatives**: If one spot is clearly the best and nothing else is close, just recommend that one spot. Alternatives are for genuinely close calls, not for padding every session window with options.
8. **Rate it** on a 1-10 scale:
   - 1-3: Barely worth paddling out
   - 4-5: Fun if you're keen, nothing special
   - 6-7: Good session, worth the effort
   - 8-9: Excellent, don't miss it
   - 10: Epic, once-in-a-trip conditions
9. **Note any caveats** (crowd risk, drive time, hazards)

### Report structure

For each day (8-day window: today + next 7 days), use the **day type** from the location map to determine the format:

#### For `surf` / `home` days — full forecast:

```
## [Day, Date] — [Location name]

**Overview**: [1-2 sentence summary of the day — overall vibe, dominant swell, wind trend]

### Dawn Patrol (5:30am - 8:00am)
**Best spot**: [Name] ([drive time] drive)
**Also worth checking**: [Nearby spot B] and [Nearby spot C] — [brief reason, e.g. "if the crowd's too much at Snapper, Greenmount handles the same swell with fewer people"]
**Check the cams**: [Specific camera suggestion, e.g. "Pull up the Snapper and D-Bah cams before you leave — if Snapper's packed, head straight to D-Bah"]
**Rating**: [X/10]
**Why**: [2-3 sentences explaining conditions and why this spot works]
**Surf**: [height range], [swell details]
**Wind**: [speed] [direction] ([offshore/onshore/cross])
**Tide**: [height]m [rising/falling]

### Pre-lunch (9:00am - 12:00pm)
[same format]

### Afternoon (12:00pm - 4:00pm)
[same format]

### Evening Session (4:00pm - 6:30pm)
[same format]

### Best advice for the day
[Your honest recommendation — which session to prioritize, whether to drive far or stay local, etc.]
```

#### For `wave-pool` days — minimal card:

```
## [Day, Date] — [Location] (Wave Pool)

**Sessions**: [Times from itinerary, e.g. "Evening session + 3pm Sunday"]
**Weather**: [Temperature, wind if outdoor pool]
**Notes**: [Any relevant notes]
```

#### For `travel` days — rest card:

```
## [Day, Date] — Travelling

[Route/notes from itinerary, e.g. "Melbourne → Portugal via Dubai"]
No surf today.
```

#### For `rest` days — rest card:

```
## [Day, Date] — Rest Day

[Optional notes]
```

**Multi-location transitions:** When the location changes between consecutive days, add a brief transition note in the day header or overview — e.g. "Last day on the Gold Coast before flying to Melbourne tomorrow." This helps the surfer mentally prepare for the transition.

**Note on "Also worth checking" and "Check the cams"**: These lines are OPTIONAL — only include them when there are genuine alternatives in the same area. If one spot is the clear winner, skip these lines entirely. A clean single recommendation is better than forced alternatives.

### Tone

Write like a knowledgeable local surf forecaster talking to a mate. Be direct, honest, and practical. Don't hype mediocre conditions. If it's flat, say it's flat. If there's a once-in-a-trip swell coming, get excited about it.

## Step 5: Update the HTML dashboard

Read the current `index.html` from the GitHub Pages repo. The local repo path is stored in `current_situation.md` under System Notes (currently `~/code/personal_surf_forecasting_claude`, mounted in Cowork at `/sessions/.../mnt/personal_surf_forecasting_claude`). **Read and write `index.html` using the normal mounted folder path** — do NOT use Desktop Commander for file operations.

**After writing the file, commit and push using the GitHub token (git only):**
Run the following in Bash from the mounted workspace folder:
```bash
cd /sessions/.../mnt/personal_surf_forecasting_claude && \
  git config user.email "rvanloo@hexarad.com" && \
  git config user.name "Roman Van Loo" && \
  git add -A && \
  git commit -m "Forecast update: [DATE]" && \
  GITHUB_TOKEN=$(cat .github_token | tr -d '[:space:]') && \
  git push https://${GITHUB_TOKEN}@github.com/RomanVanLoo/personal_surf_forecasting_claude.git HEAD:master
```
The `.github_token` file contains a fine-grained personal access token scoped to this repo only (Contents: read/write). It's already in `.gitignore` and will never be committed. This replaces the old Desktop Commander approach — no host-level tool needed.

The HTML should be a single self-contained file (no external dependencies except CDN-hosted CSS/fonts) that works as a mobile-friendly dashboard showing:
- Last updated timestamp
- Chrome connection status (warning banner if disconnected)
- Current location and trip context (may show multiple locations if the 8-day window spans a transition)
- 8-day forecast (today + next 7 days) with session windows, spot recommendations, alternatives, and camera check suggestions
- **Multi-location awareness in day chips/tabs**: Each day chip should show the location name if it differs from the previous day (e.g. "THU 26 — Gold Coast" → "SAT 28 — Melbourne" → "THU 2 — Lisbon"). Same-location consecutive days can omit the label after the first. Travel/rest days should show a distinct muted style with "Travelling" or "Rest" label.
- Day tabs or navigation to switch between days
- **Day type-specific cards**: `surf`/`home` days get full session cards. `wave-pool` days get a simple info card with session times. `travel`/`rest` days get a minimal rest card.
- A "days to watch" highlight section for notable days beyond the 8-day window (if any)
- **Data confidence indicators**: For each forecast region used, show whether a human forecaster's written report was available. Regions with written reports should display a small badge like "Forecaster verified — [Name]" to indicate enhanced accuracy. Regions without written reports should note "Numerical data only" so the user knows the analysis is purely algorithmic. This distinction matters because human forecasters account for local nuances (sand movement, rip patterns, local wind effects) that numerical models miss.

**Housekeeping — remove past dates:** Every time the dashboard is updated, remove any day tabs and session cards for dates that are now in the past. The dashboard should only ever show today and future days. Don't accumulate stale forecasts — if it's Wednesday, Monday and Tuesday should be gone. This keeps the dashboard clean and ensures the user always sees current information first.

**Dashboard design reference:** The approved HTML design uses a dark theme (bg: #0a0f1a, cards: #141b2d, accent: #3b82f6), mobile-first layout (max-width 680px), day tabs that switch content, session cards with colour-coded left borders (red=1-2, orange=3, yellow=4-5, green=6-7, cyan=8+), a conditions grid (surf/wind/tide) inside each session, girlfriend/partner-friendly tags, an advice card per day, a "Days to Watch" section with red-bordered cards for big swells, and a Data Sources section showing forecaster names. Read the existing `index.html` before regenerating — preserve the design unless the user requests changes.

If no GitHub repo exists yet, generate the HTML file and save it to the workspace for the user to set up.

## Step 6: Update the forecast log

After a successful forecast run, append an entry to the Update Log in `current_situation.md`:
```
- [DATE]: Forecast run — [X] regions fetched, Chrome [connected/failed], dashboard updated
```
This helps track when forecasts last ran and whether there were any issues.

## Error handling

- **Chrome not available**: Generate report from cached/stale data, flag prominently
- **Surfline layout changed**: If page scraping fails, log what went wrong and try alternative selectors
- **New location not in database**: Trigger research flow (Step 2)
- **No surf spots match conditions**: Say so honestly — "Stay on the couch today, nothing worth paddling out for"
