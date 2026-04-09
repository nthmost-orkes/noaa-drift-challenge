# NOAA Coastal Debris Drift Prediction Challenge

## Background & Motivation

On the morning of March 15th, 2024, the container ship *MV Pacific Wanderer* lost 47 shipping containers approximately 2.3 nautical miles west of the Golden Gate Bridge during heavy swells. The Coast Guard immediately initiated search and rescue operations, but the question on everyone's mind was: **where will the debris end up?**

This scenario, while fictional, represents a real class of problems that maritime authorities, environmental response teams, and search-and-rescue coordinators face regularly. The movement of floating objects in coastal waters is governed by a complex interplay of tidal currents, wind-driven surface currents, and the Coriolis effect. While full oceanographic modeling requires sophisticated tools like GNOME (General NOAA Operational Modeling Environment), a surprising amount of useful prediction can be done using publicly available tidal current data.

Your challenge: build a working debris drift prediction system using NOAA's Tides and Currents API.

---

## The Challenge

Given:
- An **incident location** (latitude, longitude) in a coastal region with NOAA current prediction stations
- An **incident time** (datetime)
- A **prediction window** (e.g., 24 or 48 hours)

Produce:
- A time-series of **predicted debris positions** at regular intervals (e.g., hourly)
- A visualization or clear textual output showing the drift trajectory
- An explanation of your methodology and its limitations

### Evaluation Criteria

Your solution will be evaluated on:

1. **Correctness** — Does your code actually call the NOAA API and process real data?
2. **Reasoning** — Do you understand and account for the limitations of your approach?
3. **Code Quality** — Is your solution well-structured and readable?
4. **Edge Case Handling** — What happens at API boundaries, missing data, etc.?

---

## NOAA CO-OPS API Overview

The Center for Operational Oceanographic Products and Services (CO-OPS) maintains a comprehensive API for accessing tidal and current data. The API is documented at:

**Data API**: https://api.tidesandcurrents.noaa.gov/api/prod/
**Metadata API**: https://api.tidesandcurrents.noaa.gov/mdapi/prod/

### A Brief History of Tidal Prediction

Before diving into the API specifics, it's worth understanding the remarkable history of tidal prediction. The first mechanical tide-predicting machine was built by William Thomson (Lord Kelvin) in 1872. These analog computers used systems of pulleys and gears to sum harmonic constituents — the periodic components that make up tidal motion. The U.S. Coast and Geodetic Survey operated mechanical tide predictors until 1965, when they were finally replaced by electronic computers.

Today, NOAA maintains a network of over 200 water level stations and numerous current measurement stations. The harmonic constants derived from long-term observations at these stations allow predictions to be made years in advance with remarkable accuracy. However, predictions don't account for meteorological effects — storm surge, sustained winds, and atmospheric pressure changes can cause significant deviations from predicted values.

### Understanding Tidal Currents vs. Water Levels

A common misconception is that tidal currents simply flow "in" during rising tide and "out" during falling tide. The reality is more nuanced:

1. **Slack water** (zero current) does not necessarily occur at high or low tide
2. Maximum current often occurs midway between high and low water
3. In many locations, the flood current flows for a different duration than the ebb current
4. Rotary currents (common in open water) continuously change direction rather than simply reversing

The phase relationship between water levels and currents varies by location and depends on the geometry of the coastline, bottom topography, and the dominant tidal constituents.

### Station Types and Their Uses

NOAA maintains several types of stations relevant to this challenge:

**Water Level Stations** measure the actual height of water relative to a datum (reference level). These stations provide:
- Real-time observations (preliminary data)
- Verified historical data (quality-controlled)
- Tide predictions (based on harmonic analysis)

**Current Stations** measure water velocity and direction. These come in two varieties:

1. **PORTS Stations** (Physical Oceanographic Real-Time System) — Permanent installations in major harbors that provide real-time current measurements. These stations use acoustic Doppler current profilers (ADCPs) that measure currents at multiple depths.

2. **Survey Stations** — Temporary deployments for data collection, typically lasting weeks to months. Historical data from these stations may be available but represents past conditions.

**Current Prediction Stations** — Locations where tidal current predictions are available based on harmonic analysis. These predictions represent the astronomically-driven component of currents only.

### API Request Structure

The Data API uses a RESTful structure with the following base URL:

```
https://api.tidesandcurrents.noaa.gov/api/prod/datagetter
```

Required parameters vary by product type, but typically include:

| Parameter | Description | Example |
|-----------|-------------|---------|
| `station` | Station ID (alphanumeric) | `9414290` or `SFB1203` |
| `product` | Data type requested | `predictions`, `currents_predictions` |
| `begin_date` | Start of date range | `20240315` |
| `end_date` | End of date range | `20240317` |
| `format` | Output format | `json`, `xml`, `csv` |

**Important Date/Time Notes**: 
- Dates can be formatted as `yyyyMMdd`, `yyyyMMdd HH:mm`, `MM/dd/yyyy`, or `MM/dd/yyyy HH:mm`
- The API also accepts relative date parameters: `date=today`, `date=latest`, `date=recent`
- The `range` parameter specifies hours (for most products) or years (for daily max/min)

### Data Products Relevant to Drift Prediction

For debris drift prediction, you'll primarily need **current predictions**. Here's what's available:

#### `currents_predictions`

Tidal current predictions for harmonic current prediction stations.

**Key parameters:**
- `interval` — Prediction interval. Valid values: `1`, `6`, `10`, `30`, `60` (minutes), `h` (hourly), or `MAX_SLACK` (maximum flood/ebb and slack times only)
- `bin` — Depth bin number (required for most stations)
- `vel_type` — Either `default` (flood/ebb directions) or `speed_dir` (2D speed and direction)

**Critical limitation**: Harmonic current prediction stations can provide predictions on any interval. **Subordinate stations can ONLY provide `MAX_SLACK` predictions** — attempting to request interval predictions from a subordinate station will fail.

**Data length limits:**
- `MAX_SLACK` predictions: 1 year maximum
- All other intervals: 1 month maximum

#### `currents`

Real-time or historical measured currents from PORTS or survey stations.

**Key parameters:**
- `bin` — Required for most stations. Use `bin=0` to get all bins (limited to 7 days)
- `time_zone` — `gmt`, `lst` (local standard), or `lst_ldt` (local with daylight saving)

**Note**: Real-time currents represent *actual* conditions, including meteorological effects. Predictions represent *astronomical* tides only.

### Understanding Bins

Current data is collected at multiple depths. Each depth has a "bin number." The bin numbering scheme varies:

- For **PORTS stations**, a default bin is pre-selected and the `bin` parameter is optional
- For **survey stations**, you must specify a bin
- Using `bin=0` returns all bins (with a 7-day data limit)

To find available bins for a station, use the Metadata API:

```
https://api.tidesandcurrents.noaa.gov/mdapi/prod/webapi/stations/{station_id}/bins.json
```

The response includes:
- `nbr_of_bins` — Total number of bins
- `binList` — Array of bins with depth information
- `depth` — Depth of each bin (in feet or meters, depending on `units` parameter)

For drift prediction of floating debris, **you typically want the shallowest bin** (surface currents). However, be aware that some stations measure from the bottom up, so "bin 1" might be the deepest.

### Current Prediction Station Types

Current prediction stations come in three types, indicated by the `type` field in the metadata:

| Type | Description | Available Intervals |
|------|-------------|---------------------|
| `H` | Harmonic — predictions based on harmonic analysis | All intervals (1, 6, 10, 30, 60 min, hourly) |
| `S` | Subordinate — predictions derived from a reference station | `MAX_SLACK` only |
| `W` | Weak and Variable — currents too weak for reliable prediction | `MAX_SLACK` only |

**This is crucial**: If you request `interval=h` from a subordinate station, the API will return an error. You must either use `MAX_SLACK` or find an alternative harmonic station.

### Sample API Calls

**Get current predictions (hourly) for a harmonic station:**
```
https://api.tidesandcurrents.noaa.gov/api/prod/datagetter?station=SFB1203&product=currents_predictions&begin_date=20240315&end_date=20240316&interval=h&units=english&time_zone=gmt&format=json
```

**Get current predictions (max/slack) for any station type:**
```
https://api.tidesandcurrents.noaa.gov/api/prod/datagetter?station=PCT1291&product=currents_predictions&begin_date=20240315&end_date=20240316&interval=MAX_SLACK&units=english&time_zone=gmt&format=json
```

**Get metadata for current prediction stations in a region:**
```
https://api.tidesandcurrents.noaa.gov/mdapi/prod/webapi/stations.json?type=currentpredictions
```

### Response Format

For `currents_predictions` with `interval=h` (hourly), the JSON response looks like:

```json
{
  "current_predictions": {
    "units": "knots",
    "cp": [
      {
        "t": "2024-03-15 00:00",
        "Velocity_Major": 1.23,
        "meanEbbDir": 245,
        "meanFloodDir": 65,
        "Bin": 1
      },
      ...
    ]
  }
}
```

For `MAX_SLACK` interval, the response structure differs:

```json
{
  "current_predictions": {
    "units": "knots",
    "cp": [
      {
        "t": "2024-03-15 02:34",
        "type": "slack"
      },
      {
        "t": "2024-03-15 05:48",
        "Velocity_Major": -2.1,
        "type": "ebb"
      },
      ...
    ]
  }
}
```

**Important velocity conventions:**
- Positive `Velocity_Major` indicates **flood** current (flowing in `meanFloodDir` direction)
- Negative `Velocity_Major` indicates **ebb** current (flowing in `meanEbbDir` direction)
- For `vel_type=speed_dir`, you get `Speed` and `Direction` fields instead

### Finding Stations

To find current prediction stations, use the Metadata API:

```
https://api.tidesandcurrents.noaa.gov/mdapi/prod/webapi/stations.json?type=currentpredictions
```

Each station record includes:
- `id` — Station identifier (use this in API calls)
- `name` — Human-readable station name
- `lat`, `lng` — Station coordinates
- `type` — `H`, `S`, or `W` (see above)
- `depth` — Prediction depth in feet
- `currbin` — The bin number for predictions

**Pro tip**: The station ID format varies. Some examples:
- `SFB1203` — San Francisco Bay
- `ACT1016` — Anchorage, Cook Inlet
- `PCT1291` — Pacific coast

### Units and Time Zones

**Units:**
- `english` — Velocity in knots, depth in feet
- `metric` — Velocity in cm/s, depth in meters

**Time zones:**
- `gmt` — Greenwich Mean Time (UTC)
- `lst` — Local Standard Time (no daylight saving adjustment)
- `lst_ldt` — Local time with daylight saving when applicable

For computation purposes, **always use `gmt`** to avoid daylight saving time confusion.

### API Throttling

CO-OPS protects its APIs by limiting request volume. Best practices:
- Add sleep intervals between successive API calls
- Only request the data you need
- Cache responses when appropriate

No specific rate limit is published, but rapid-fire requests will be throttled.

### Error Handling

The API returns errors in the requested format:

```json
{
  "error": {
    "message": "Great Lakes stations don't have Predictions data."
  }
}
```

Common errors:
- Wrong date range (end before begin)
- Station doesn't support requested product
- Invalid parameter combinations
- Date range exceeds product limits

---

## Implementation Guidance

### Suggested Approach

1. **Find nearby stations** — Given your incident location, identify current prediction stations in the area. Consider using the Metadata API to filter by type=currentpredictions.

2. **Check station types** — Verify whether nearby stations are Harmonic (H), Subordinate (S), or Weak/Variable (W). This determines what data you can retrieve.

3. **Get predictions** — Request current predictions for your time window. For Harmonic stations, use hourly intervals. For Subordinate stations, you'll need to work with MAX_SLACK data and interpolate.

4. **Calculate drift** — Apply the predicted currents to update position over time. Remember:
   - Velocity is typically in knots (1 knot = 1.852 km/h = 1 nautical mile/hour)
   - Direction conventions matter (meteorological vs. oceanographic)
   - You're approximating — real drift involves many factors not captured in tidal predictions

### Interpolation Strategies

If you only have MAX_SLACK data (slack times and max flood/ebb), you'll need to interpolate. Common approaches:

1. **Linear interpolation** — Simple but inaccurate (currents don't change linearly)
2. **Sinusoidal interpolation** — Better approximation:
   ```
   V(t) = V_max * sin(π * (t - t_slack) / (t_max - t_slack))
   ```
3. **Use multiple stations** — Combine predictions from nearby Harmonic stations

### Direction Conventions

**Oceanographic convention** (used by NOAA currents): Direction current is flowing **TO**
- A current with direction 90° is flowing toward the east

**Meteorological convention** (used for winds): Direction wind is coming **FROM**
- A wind with direction 90° is coming from the east (blowing westward)

The NOAA currents API uses **oceanographic convention**.

### Coordinate Math

For small distances (under ~100 km), you can use a flat-Earth approximation:

```python
# Approximate meters per degree at a given latitude
meters_per_deg_lat = 111320  # constant
meters_per_deg_lon = 111320 * cos(latitude_radians)

# Convert velocity (knots) to degrees/hour
knots_to_mps = 0.514444  # meters per second
velocity_mps = velocity_knots * knots_to_mps
velocity_deg_lat = (velocity_mps * 3600) / meters_per_deg_lat
velocity_deg_lon = (velocity_mps * 3600) / meters_per_deg_lon
```

For longer distances or higher accuracy, use the haversine formula or a geodesy library.

### Known Limitations

Your solution should acknowledge:

1. **Tidal predictions only** — No wind, no waves, no residual currents
2. **Point predictions** — Real currents vary spatially; you're extrapolating from point measurements
3. **Surface layer assumption** — Debris at different depths experiences different currents
4. **No windage** — Floating objects exposed to wind experience additional drift (typically 1-3% of wind speed)
5. **Simplified physics** — Real drift modeling uses ensemble methods and accounts for uncertainty

---

## Test Scenario: San Francisco Bay

For a concrete test case, use this scenario:

**Incident Details:**
- Location: 37.8044° N, 122.4780° W (approximately 2 NM west of Golden Gate Bridge)
- Time: March 15, 2024, 14:00 UTC
- Prediction window: 24 hours

**Nearby Current Prediction Stations:**
- `SFB1203` — Golden Gate (type: H)
- `SFB1204` — Pt. Bonita to Lime Pt. (type: S)
- `SFB1202` — Off Mile Rocks (type: H)

Note: You should verify these station IDs and types using the Metadata API, as they may have changed.

**Expected behavior:**
- During ebb tide, debris should move seaward (westward)
- During flood tide, debris should move into the bay (eastward)
- Net drift depends on the tidal cycle timing relative to incident time

---

## Deliverables

Please submit:

1. **Working Python code** that:
   - Takes incident location, time, and duration as input
   - Queries the NOAA API for relevant current predictions
   - Calculates and outputs predicted debris positions
   - Handles errors gracefully

2. **A brief explanation** (can be comments in code or a separate document):
   - Your methodology
   - Assumptions and limitations
   - Ideas for improvement

3. **Sample output** from running your code on the San Francisco Bay scenario

---

## Time Budget

This challenge is designed for approximately **60 minutes**:
- 10-15 min: Understanding the API and planning
- 30-40 min: Implementation
- 10-15 min: Testing and documenting limitations

You are welcome to use any tools, documentation, or AI assistants. The goal is to demonstrate your ability to tackle an unfamiliar domain efficiently.

---

## Additional Resources

- [NOAA Tides & Currents](https://tidesandcurrents.noaa.gov/) — Main portal
- [CO-OPS Station Map](https://tidesandcurrents.noaa.gov/map/) — Interactive station finder  
- [API Builder Tool](https://tidesandcurrents.noaa.gov/api-helper/url-generator.html) — Interactive query builder
- [Response Help Page](https://api.tidesandcurrents.noaa.gov/api/prod/responseHelp.html) — Field descriptions

---

## Questions?

If you have questions about the challenge requirements (not implementation questions), please ask your interviewer.

Good luck!
