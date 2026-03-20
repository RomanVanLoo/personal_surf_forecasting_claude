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

## Step 1: Read the current situation

Read `current_situation.md` from the user's `surf_forecasting/` workspace folder. Extract:
- Current location and how long they're staying
- Max driving radius
- All surfer profiles (preferences, skill levels, wave size limits, crowd tolerance)
- Trip context (working? flexible schedule?)
- Any special notes
- The `last_updated_at` timestamp at the top of the file

If the file doesn't exist or is empty, ask the user where they are and what they're looking for. Use the `update-situation` skill pattern to create it.

## Step 2: Check spot database coverage & freshness

Read `best_conditions_for_surf_spots.json` from the same folder. Check the `metadata.region`, `metadata.base_location`, and `metadata.created_at` fields.

### 2a. Staleness check — has the situation changed since the database was built?

Compare the `last_updated_at` timestamp from `current_situation.md` against the `created_at` timestamp from the JSON metadata.

**If `last_updated_at` is AFTER `created_at`:** The surfer's situation has changed since the spot database was last built. This means the database may be stale — the surfer may have moved, their crew may have changed, or their preferences may have shifted. **Treat this as a full rebuild trigger:**

1. Notify the user: "Your situation was updated on [last_updated_at] but the spot database was built on [created_at]. I need to rebuild it to match your current situation."
2. Proceed to the database rebuild flow below (Step 2b).

**If `last_updated_at` is BEFORE or EQUAL to `created_at`:** The database is fresh. Proceed to Step 2b's coverage check only.

### 2b. Coverage check — does the database cover the current location?

Compare the location from `current_situation.md` against the region in the JSON. If the surfer has moved to a new area that isn't covered (or if the staleness check triggered a rebuild):

1. Notify the user: "You've moved to [new location] but the spot database still covers [old region]. I need to rebuild it for your new area."
2. Clear the existing spots and forecast_regions from the JSON
3. Research surf spots within the driving radius of the new location using web search
4. For each spot, gather: name, Surfline ID, swell/wind/tide preferences, crowd level, skill level, wave type, character, hazards, drive time
5. Group spots into forecast regions, each with a reference Surfline spot
6. Tag each spot with its forecast_region
7. **Update `metadata.created_at`** to the current ISO 8601 timestamp (e.g. `2026-03-16T14:30:00+10:00`)
8. Write the updated JSON

This research step is the most time-intensive part. It only needs to happen when the surfer changes location to an uncovered area, or when the situation file is newer than the database.

If the database already covers the current location and passes the staleness check, skip to Step 3.

## Step 3: Fetch Surfline forecast data

This step requires Chrome browser automation via Claude in Chrome.

### Chrome connection notes

- **Always use the "Private" Chrome window** — the user may have multiple Chrome windows open. Specify the "Private" window when using Chrome tools to avoid conflicts.
- **Preferred data extraction method: `get_page_text`** — Surfline pages are heavy (video players, ads, interactive charts). Screenshots via the `computer` tool frequently timeout. Always use `get_page_text` as the primary extraction method — it's faster, more reliable, and returns more complete data including the full hourly forecast tables.
- **URL trick: append `?view=table`** to the Surfline spot URL to load the tabular forecast view directly. This gives you the hourly breakdown without needing to click through tabs.

### 3a. Determine which regions to fetch

Read the `forecast_regions` from the JSON metadata. The strategy field tells you which regions to always fetch and which are conditional.

For the current trip, evaluate:
- **Always fetch**: Core regions near the base location
- **Conditionally fetch**: Distant regions (2hr+ drive) — only if the general swell pattern suggests exceptional conditions there

### 3b. Navigate to each reference spot on Surfline

For each region to fetch:

1. Navigate to the reference spot's Surfline URL
2. The surfer should already be logged in via cookies in Chrome
3. Click on "Today" to expand the full daily view with the written forecaster report
4. Extract using `get_page_text`:
   - The written forecaster report (by name, e.g. "Lachlan Perris") — **NOTE: Not all regions have written forecaster reports.** Some regions (e.g. Portugal, parts of Asia/Pacific) only have numerical data and no human-written analysis. If no written report is present, proceed with the numerical data only and flag this in the output.
   - Current conditions (rating, surf height, swell, wind, tide, temperature)
   - "Days to Watch" section (may also be absent in some regions)
5. Click on "Tomorrow" and extract the same data
6. Click on the day after tomorrow and extract the same data
7. Also capture the hourly forecast table for all 3 days (surf height, primary swell, secondary swell, wind speed+direction, wave energy, consistency, weather, tide times)

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

Read `observations.json` from the workspace folder. Filter for observations from the **last 3 days** that are relevant to the current forecast period.

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
4. **Which spots light up under these conditions** — cross-reference the forecast data against each spot's ideal conditions in the JSON
5. **Filter for ALL surfer profiles** from `current_situation.md` — for each person in the group, check their wave size limits, skill level, and crowd tolerance. Exclude spots that don't work for everyone who's surfing that day. Tag each session with who it's suitable for (e.g. "Girlfriend-friendly" or "Roman only — too heavy for Cato"). Use the `girlfriend_friendly` field in the spot database as a starting point, but also cross-reference against each person's specific limits.
6. **Pick the best spot for the window** and explain why
7. **Rate it** on a 1-10 scale:
   - 1-3: Barely worth paddling out
   - 4-5: Fun if you're keen, nothing special
   - 6-7: Good session, worth the effort
   - 8-9: Excellent, don't miss it
   - 10: Epic, once-in-a-trip conditions
8. **Note any caveats** (crowd risk, drive time, hazards)

### Report structure

For each day (today, tomorrow, day after):

```
## [Day, Date]

**Overview**: [1-2 sentence summary of the day — overall vibe, dominant swell, wind trend]

### Dawn Patrol (5:30am - 8:00am)
**Best spot**: [Name] ([drive time] drive)
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
[Your honest recommendation — which session to prioritize, whether to drive far or stay local, whether the girlfriend will enjoy it, etc.]
```

### Tone

Write like a knowledgeable local surf forecaster talking to a mate. Be direct, honest, and practical. Don't hype mediocre conditions. If it's flat, say it's flat. If there's a once-in-a-trip swell coming, get excited about it.

## Step 5: Update the HTML dashboard

Read the current `index.html` from the GitHub Pages repo. The local repo path is stored in `current_situation.md` under System Notes (currently `~/code/personal_surf_forecasting_claude`, mounted in Cowork at `/sessions/.../mnt/personal_surf_forecasting_claude`). **Read and write `index.html` using the normal mounted folder path** — do NOT use Desktop Commander for file operations.

**After writing the file, commit and push using Desktop Commander (git only):**
Use `mcp__Desktop_Commander__start_process` with `timeout_ms: 30000` to run:
`cd /Users/romanvanloo/code/personal_surf_forecasting_claude && git add -A && git commit -m "Forecast update: [DATE]" && git push`
This pushes directly to GitHub Pages via the user's local git credentials. Desktop Commander is ONLY used for this git step — not for reading or writing files.

The HTML should be a single self-contained file (no external dependencies except CDN-hosted CSS/fonts) that works as a mobile-friendly dashboard showing:
- Last updated timestamp
- Chrome connection status (warning banner if disconnected)
- Current location and trip dates
- Today's forecast with session windows and spot recommendations
- Tomorrow and day-after forecasts (collapsible)
- A "days to watch" highlight section
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
