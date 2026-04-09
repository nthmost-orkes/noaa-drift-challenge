# NOAA Coastal Debris Drift Prediction Challenge

## Background & Motivation

On the morning of March 10th, 2026, the container ship *MV Pacific Wanderer* lost 47 shipping containers approximately 2.3 nautical miles west of the Golden Gate Bridge during heavy swells. The Coast Guard immediately initiated search and rescue operations, but the question on everyone's mind was: **where will the debris end up?**

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

## NOAA CO-OPS API Reference

The Center for Operational Oceanographic Products and Services (CO-OPS) maintains a comprehensive API for accessing tidal and current data. The API is documented at:

**Data API**: https://api.tidesandcurrents.noaa.gov/api/prod/

**Metadata API**: https://api.tidesandcurrents.noaa.gov/mdapi/prod/

### Historical Context

Before diving into the API specifics, it's worth understanding the remarkable history of tidal prediction. The first mechanical tide-predicting machine was built by William Thomson (Lord Kelvin) in 1872. These analog computers used systems of pulleys and gears to sum harmonic constituents — the periodic components that make up tidal motion. The U.S. Coast and Geodetic Survey operated mechanical tide predictors until 1965, when they were finally replaced by electronic computers.

Today, NOAA maintains a network of over 200 water level stations and numerous current measurement stations. The harmonic constants derived from long-term observations at these stations allow predictions to be made years in advance with remarkable accuracy. However, predictions don't account for meteorological effects — storm surge, sustained winds, and atmospheric pressure changes can cause significant deviations from predicted values.

The distinction between harmonic and subordinate stations dates back to the era of tide tables. Reference (harmonic) stations have sufficient observational data for independent harmonic analysis, while subordinate stations have their predictions derived by applying time and height corrections to a nearby reference station. This distinction persists in the modern API and affects what data products are available.

### Understanding Tidal Currents vs. Water Levels

A common misconception is that tidal currents simply flow "in" during rising tide and "out" during falling tide. The reality is more nuanced. Slack water (zero current) does not necessarily occur at high or low tide. Maximum current often occurs midway between high and low water. In many locations, the flood current flows for a different duration than the ebb current. Rotary currents (common in open water) continuously change direction rather than simply reversing.

The phase relationship between water levels and currents varies by location and depends on the geometry of the coastline, bottom topography, and the dominant tidal constituents. In narrow channels, currents tend to be rectilinear (reversing), while in open bays they may be rotary. The San Francisco Bay system exhibits both patterns depending on location.

### Station Types

NOAA maintains several types of stations:

**Water Level Stations** measure the actual height of water relative to a datum (reference level). These stations provide real-time observations (preliminary data), verified historical data (quality-controlled), and tide predictions (based on harmonic analysis). Water level stations are identified by 7-digit numeric IDs like `9414290` (San Francisco).

**Current Stations** measure water velocity and direction. PORTS Stations (Physical Oceanographic Real-Time System) are permanent installations in major harbors that provide real-time current measurements using acoustic Doppler current profilers (ADCPs) that measure currents at multiple depths. Survey Stations are temporary deployments for data collection, typically lasting weeks to months. Current stations use alphanumeric IDs that vary by region — for example, `cb0102` for Chesapeake Bay PORTS stations.

**Current Prediction Stations** are locations where tidal current predictions are available based on harmonic analysis. These use alphanumeric IDs with regional prefixes: `SFB` for San Francisco Bay, `ACT` for Alaska Cook Inlet, `PCT` for Pacific Coast, etc. Current prediction stations are classified by type in the metadata: type `H` indicates a harmonic station with full observational data, type `S` indicates a subordinate station whose predictions are derived from a reference station, and type `W` indicates weak and variable currents. The station type determines what prediction intervals are available through the API.

### Data API Structure

The Data API uses a RESTful structure with the base URL:

```
https://api.tidesandcurrents.noaa.gov/api/prod/datagetter
```

Parameters are passed as query string arguments. Required parameters vary by product type. Common parameters include:

| Parameter | Description | Example Values |
|-----------|-------------|----------------|
| `station` | Station ID | `9414290`, `SFB1203`, `cb0102` |
| `product` | Data type | `water_level`, `predictions`, `currents`, `currents_predictions` |
| `begin_date` | Start date | `20260310`, `20260310 14:00` |
| `end_date` | End date | `20260311`, `20260311 14:00` |
| `datum` | Reference datum | `MLLW`, `NAVD`, `STND` |
| `units` | Measurement units | `english`, `metric` |
| `time_zone` | Time reference | `gmt`, `lst`, `lst_ldt` |
| `format` | Output format | `json`, `xml`, `csv` |
| `application` | Your app name | `MyDriftApp` |

Date formats accepted: `yyyyMMdd`, `yyyyMMdd HH:mm`, `MM/dd/yyyy`, `MM/dd/yyyy HH:mm`. The API also accepts relative dates: `date=today`, `date=latest`, `date=recent`. The `range` parameter specifies hours for most products.

### Data Products

**Water Level Products:**
- `water_level` — 6-minute interval preliminary or verified data
- `hourly_height` — Verified hourly water levels
- `high_low` — Verified high/low tide times and heights
- `predictions` — Tide predictions (water level)

**Current Products:**
- `currents` — Real-time or historical measured currents
- `currents_predictions` — Tidal current predictions

**Meteorological Products:**
- `air_temperature`, `water_temperature`, `wind`, `air_pressure`, `visibility`, `humidity`

For current predictions, additional parameters control the output:

| Parameter | Description | Values |
|-----------|-------------|--------|
| `interval` | Prediction interval | `1`, `6`, `10`, `30`, `60` (minutes), `h` (hourly), `MAX_SLACK` |
| `bin` | Depth bin number | Integer, or `0` for all bins |
| `vel_type` | Velocity output type | `default`, `speed_dir` |

The `vel_type=default` returns velocity relative to flood/ebb directions, with positive values indicating flood current and negative indicating ebb. The `vel_type=speed_dir` returns 2D speed and direction. The default velocity type uses the mean flood and mean ebb directions for the station; speed_dir gives instantaneous direction which may differ from the mean directions, especially for rotary currents.

### Data Length Limits

The API enforces limits on how much data can be retrieved per request:

| Data Type | Maximum Length |
|-----------|----------------|
| 1-minute interval | 4 days |
| 6-minute interval | 1 month |
| Hourly interval | 1 year |
| High/Low data | 1 year |
| Daily means | 10 years |
| Monthly means | 200 years |
| Current predictions (MAX_SLACK) | 1 year |
| Current predictions (other intervals) | 1 month |
| All bins (bin=0) | 7 days |

### Metadata API Structure

The Metadata API provides information about stations:

```
https://api.tidesandcurrents.noaa.gov/mdapi/prod/webapi/stations.json
https://api.tidesandcurrents.noaa.gov/mdapi/prod/webapi/stations/{id}.json
https://api.tidesandcurrents.noaa.gov/mdapi/prod/webapi/stations/{id}/{resource}.json
```

Filter stations by type with `?type=`:
- `waterlevels` — Active water level stations
- `currentpredictions` — Current prediction stations  
- `currents` — Active current meter stations
- `tidepredictions` — Tide prediction stations

Resources available for individual stations:
- `details` — Installation info, time zone, chart number
- `datums` — Water level datums
- `harcon` — Harmonic constituents
- `bins` — Depth bins for current stations
- `sensors` — Installed sensors and status
- `nearby` — Nearby stations (use `?radius=N` in nautical miles)

The `expand` parameter embeds related resources in the response: `?expand=details,sensors,bins`

### Current Prediction Response Format

For `currents_predictions` with `vel_type=default`:

```json
{
  "current_predictions": {
    "units": "knots",
    "cp": [
      {
        "t": "2026-03-10 00:00",
        "Velocity_Major": 1.23,
        "meanEbbDir": 245,
        "meanFloodDir": 65,
        "Bin": 1
      }
    ]
  }
}
```

For `currents_predictions` with `interval=MAX_SLACK`:

```json
{
  "current_predictions": {
    "units": "knots",
    "cp": [
      {"t": "2026-03-10 02:34", "type": "slack"},
      {"t": "2026-03-10 05:48", "Velocity_Major": -2.1, "meanEbbDir": 245, "type": "ebb"},
      {"t": "2026-03-10 08:52", "type": "slack"},
      {"t": "2026-03-10 12:06", "Velocity_Major": 1.8, "meanFloodDir": 65, "type": "flood"}
    ]
  }
}
```

For `currents_predictions` with `vel_type=speed_dir`:

```json
{
  "current_predictions": {
    "units": "knots",
    "cp": [
      {
        "t": "2026-03-10 00:00",
        "Speed": 1.23,
        "Direction": 67.5,
        "Bin": 1
      }
    ]
  }
}
```

### Depth Bins

Current measurements and predictions are made at specific depths. Each depth is assigned a bin number. The bin configuration varies by station and instrument. Some stations measure from surface down (bin 1 = shallowest), others from bottom up (bin 1 = deepest). The metadata API's `bins` resource provides bin depth information:

```
https://api.tidesandcurrents.noaa.gov/mdapi/prod/webapi/stations/{id}/bins.json
```

For PORTS current stations, a default bin is pre-selected and the bin parameter is optional. For other current stations, bin must be specified. Using `bin=0` retrieves all bins but is limited to 7 days of data.

For current prediction stations, the `currbin` field in the station metadata indicates which bin is used for predictions, and the `depth` field gives the prediction depth.

### Units and Conventions

**Units:**
- `english` — Velocity in knots, depth in feet, temperature in °F
- `metric` — Velocity in cm/s, depth in meters, temperature in °C

**Time zones:**
- `gmt` — Greenwich Mean Time / UTC
- `lst` — Local Standard Time (no daylight saving)
- `lst_ldt` — Local time with daylight saving adjustment

**Direction convention:** NOAA current directions use oceanographic convention — the direction the current is flowing TO, measured clockwise from true north. A current with direction 90° flows toward the east. This differs from meteorological convention where wind direction indicates where wind comes FROM.

**Velocity sign convention:** With `vel_type=default`, `Velocity_Major` is positive for flood current (toward `meanFloodDir`) and negative for ebb current (toward `meanEbbDir`).

### Error Responses

Errors are returned in the requested format:

```json
{
  "error": {
    "message": "No data was found. This product may not be offered at this station at the requested time."
  }
}
```

Common error conditions: invalid station ID, product not available for station type, date range exceeds limits, invalid parameter combinations, station has no data for requested date range.

### Rate Limiting

CO-OPS may throttle requests during high load. Best practices: add delays between successive API calls, request only needed data ranges, cache responses when appropriate. No specific rate limit is published.

---

## Test Scenario

**Incident Details:**
- Location: 37.8044° N, 122.4780° W (approximately 2 NM west of Golden Gate Bridge)
- Time: March 10, 2026, 14:00 UTC
- Prediction window: 24 hours

You should explore the NOAA API to find relevant stations near this location and determine what data is available.

### Fallback Data

If the NOAA API is unavailable during your session, `fallback_data.zip` contains cached API responses for the test scenario:
- Station metadata and current predictions for nearby stations
- Data covering March 10-12, 2026

Extract and use this data only if you cannot reach the live API.

---

## Deliverables

### Core Deliverable

**Working code** (any language) that:
- Takes incident location (lat/lon), time, and duration as input
- Queries the NOAA API for relevant current predictions
- Calculates and outputs predicted debris positions at regular intervals
- Handles errors gracefully

This can be a command-line script, a function you call from a REPL, or any other format that demonstrates working functionality.

### Stretch Goal: Web Interface

If time permits, wrap your solution in a **local web application** that allows a user to:

1. **Input** the last known coordinates of a vessel (latitude, longitude), the incident time, and a prediction window (hours)
2. **Output** a list of probable debris locations over time, showing the predicted drift trajectory

You may use any web framework or approach. The bar here is functional, not polished.

### Design Consideration: Production Readiness

Imagine this system will be deployed as a **high-availability service for Coast Guard emergency dispatchers**. When a distress call comes in, responders need drift predictions *immediately* — every minute of delay affects search area calculations and resource deployment.

In your explanation or discussion, address:
- How would you architect this for sub-second response times?
- What would you cache, pre-compute, or optimize?
- How would you handle NOAA API outages or slowdowns?
- What are the failure modes and how would you mitigate them?

You don't need to implement production infrastructure, but we want to see that you're thinking beyond "it works on my laptop."

### Also submit

- **A brief explanation** (can be comments in code or a separate document):
  - Your methodology for calculating drift
  - Assumptions and limitations
  - How you'd approach the production readiness concerns above

- **Sample output** from running your code on the test scenario

---

## Time Budget

This challenge is designed for approximately **60 minutes**:
- 10-15 min: Understanding the problem and exploring the API
- 30-40 min: Implementation
- 10-15 min: Testing and documenting limitations

You are welcome to use any tools, documentation, or AI assistants. The goal is to demonstrate your ability to tackle an unfamiliar domain efficiently.

---

## Additional Resources

- [NOAA Tides & Currents](https://tidesandcurrents.noaa.gov/) — Main portal
- [CO-OPS Station Map](https://tidesandcurrents.noaa.gov/map/) — Interactive station finder  
- [API Builder Tool](https://tidesandcurrents.noaa.gov/api-helper/url-generator.html) — Interactive query builder
- [Data API Documentation](https://api.tidesandcurrents.noaa.gov/api/prod/) — Full API reference
- [Metadata API Documentation](https://api.tidesandcurrents.noaa.gov/mdapi/prod/) — Station metadata reference

---

## Questions?

If you have questions about the challenge requirements (not implementation questions), please ask your interviewer.

Good luck!
