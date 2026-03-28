# Minnetonka Drone Tracker — Project Bible

## Overview
An iOS app that tracks public safety drone flights in Minnetonka, MN. Think FlightRadar24 for the city's Drone as First Responder (DFR) program. Built by CJC Digital LLC.

---

## Product

### What It Does
- Displays every public safety drone flight on an interactive map
- Animated cinematic playback of complete flight paths (station → scene → station)
- Push notifications when drones operate near saved locations
- Satellite and 3D map views
- Station profiles with drone capabilities
- Flight filtering by date, purpose, and station

### Monetization
| Tier | Price | Features |
|------|-------|----------|
| Free | $0 | Browse flights, map view, basic details |
| Plus | $4.99 one-time | Animated playback, full history, satellite/3D maps, filtering |
| Premium | $9.99/month | Everything + push alerts for up to 10 saved locations |

### ICP (Ideal Customer Profiles)
**ICP 1 — The Data Nerd:** Listens to police scanners, tracks flights on FlightRadar24, loves having access to public data. Wants to explore, zoom in, watch replays. Buys Plus for the playback. Upgrades to Premium because they want to know everything.

**ICP 2 — The Cautious Community Member:** Parent, homeowner, concerned about safety. Less curious, more nervous. Wants to know if something is happening near their home or kids' school. Buys Premium for the alerts. The "Active Scene near My Home" notification is the killer feature for this person.

---

## Technical Architecture

### Data Pipeline
```
City of Minnetonka → ArcGIS Feature Service (public, free, no auth)
                          ↓
                    Supabase Edge Function (polls every 10 min)
                          ↓
                    PostgreSQL + PostGIS (94+ flights stored)
                          ↓
              ┌───────────┴───────────┐
              ↓                       ↓
        iOS App (reads)         Geofence Checker
                                      ↓
                                APNs Push Notifications
```

### Data Source
- **URL:** `https://services7.arcgis.com/mnhQTdIYDA7UoY2l/arcgis/rest/services/a5d35662-d210-4194-b2c4-7e9332fc2732-production/FeatureServer/0`
- **Type:** Public ArcGIS Feature Service (no API key needed)
- **Fields:** flight_id, takeoff (epoch ms), landing (epoch ms), flight_purpose, vehicle_serial, dock_serial, external_id, description, geometry (polyline paths)
- **Discovery:** The Skydio Cloud dashboard at cloud.skydio.com makes a GraphQL call with header `Transparency-Dashboard-Path`, which returns an ArcGIS dashboard URL. The underlying Feature Service is fully public.
- **Update frequency:** Flights appear within hours of mission completion (not real-time)
- **Max records per query:** 2,000

### Key Data Insight
The public flight data only shows the **on-scene observation portion** of each flight. Transit from fire station to scene and back is NOT included. We reconstruct the full path by:
1. Inferring the departure station (nearest to first data point)
2. Adding synthetic transit legs (interpolated straight line)
3. The drone always returns to the same station it launched from

ArcGIS stores paths as multi-segment polylines (50+ segments per flight) that are **interleaved** — outbound and return data mixed together. Our PathSmoother detects geographic clusters and removes boomerang oscillations.

### Backend — Supabase
- **Project:** zkohaurrecakwbdgopam.supabase.co
- **Database:** PostgreSQL 17 + PostGIS
- **Tables:** flights, vehicles, users, geofences, alert_filters, notifications_log, purchases, telemetry_points
- **Edge Functions:** poll-flights, check-geofences, send-alerts, health-check
- **Auth:** Sign in with Apple (configured)
- **Cron:** pg_cron polls ArcGIS every 10 minutes

### iOS App — SwiftUI
- **Min iOS:** 17.0
- **Framework:** SwiftUI + MapKit + StoreKit 2 + SwiftData
- **Architecture:** MVVM with @Observable
- **Bundle ID:** com.cjcdigital.dronetracker
- **Team ID:** 9J8XV6HFTY

### Alert System
Two alert categories:
1. **Flyover:** Flight path intersects user's geofence circle (PostGIS `ST_Intersects`)
2. **Active Scene:** Scene centroid + 500m buffer overlaps geofence (`ST_DWithin`)

Training flights (BCA Reason Code 9) are excluded from all alerts.

Severity levels:
- **High:** Criminal Activity, Fire/Hazmat
- **Medium:** Missing Person, Active Call
- **Low:** Other non-training

Push notifications sent in parallel batches of 50, rounds of 500 with 200ms delay.

---

## Features Built

### Map Tab
- Flat MapKit map of Minnetonka with all 4 fire station pins
- Flight paths colored by purpose type
- Data freshness pill (green/yellow/orange based on age)
- Tap station → station profile sheet
- Filter by date range and station

### Activity Tab
- Chronological flight list grouped by day (Today / Yesterday / date)
- Search by purpose, station name
- Station filter chips
- Call type icons (graduation cap, shield, flame, phone, person)
- Relative timestamps ("17h ago")
- "from Station 2 · 18m on scene" subtitle
- Pull to refresh

### Flight Detail (Full Screen)
- **Playback mode:** Cinematic animated replay (16s at 1x speed)
  - Launch overlay: "DRONE LAUNCH" + station name + mission purpose
  - Flight animation with directional drone icon
  - Pulsing observation ring during on-scene phase
  - Landing overlay: "RETURNING TO BASE" + "Mission Complete"
  - Scrubber slider, speed control (0.5x-4x)
  - Phase indicator pill (En Route / On Scene / Returning)
- **Map mode:** Interactive pinch/zoom/rotate/tilt
  - Standard, Satellite, and 3D map styles
  - Full flight path rendered
- Scene zone circle (dynamically sized to observation footprint)
- Station pin labeled "Launch"
- Share button (text summary)
- Deployed from card with station image, address, drone count
- Scene Arrival / Departure times with relative timestamps

### Alerts Tab
- Alert type explainer (Flyover + Active Scene)
- Geofence list with radius + 500m buffer info
- Add/delete geofences
- Alert history with severity badges and distance

### Settings Tab
- Push notification status (Enabled/Disabled/Not Set Up)
- "Open Notification Settings" button if denied
- Alert zone management
- Subscription tier badge
- Data source info (ArcGIS, 10 min refresh, hours freshness)
- Links to DFR program and official dashboard
- Privacy policy and terms of use links

### Station Profiles
- Gradient header with station name and nickname
- Address + 24/7 vs Daytime staffing badge
- Drone count, model (Skydio X10), deploy time (40 sec)
- Capability chips: 4K/60fps, Thermal, Night Ops, Speaker, Spotlight, Parachute

---

## Minnetonka DFR Program Facts
- **Launched:** August 25, 2025
- **Cost:** $265,000/year
- **Drones:** 6 Skydio X10 across 4 fire stations
- **Deploy time:** 40 seconds from launch to airborne
- **Coverage:** Any point in the city within ~90 seconds
- **Statute:** Minnesota Statutes § 626.19

### Fire Stations
| Station | Nickname | Address | Drones | Staffing |
|---------|----------|---------|--------|----------|
| 1 | Civic Center Park | 14550 Minnetonka Blvd | 2 | 24/7 |
| 2 | Oak Knoll | 1815 Hopkins Crossroad | 2 | Daytime |
| 3 | Lone Lake | 5700 Rowland Road | 1 | 24/7 |
| 4 | Covington | 17125 Excelsior Blvd | 1 | Daytime |

### Flight Purposes in Data
| Purpose | BCA Code | Count | Severity |
|---------|----------|-------|----------|
| Reasonable Suspicion of Criminal Activity | 7 | 33 | High |
| Officer Training or PR | 9 | 31 | Excluded |
| Emergency Risk of Death/Bodily Harm | 1 | 25 | High |
| Missing Person | 11 | 1 | Medium |
| Public Event Risk to Safety | 3 | 1 | Medium |
| Unrelated to Law Enforcement | 10 | 1 | Low |

---

## App Store Submission

### Listing
- **Name:** Minnetonka Drone Tracker
- **Subtitle:** Drone Flights Near You
- **Category:** Reference (primary), News (secondary)
- **Age Rating:** 12+
- **Bundle ID:** com.cjcdigital.dronetracker

### URLs
- **Support:** https://carterjensen.github.io/minnetonka-drone-tracker/
- **Privacy:** https://carterjensen.github.io/minnetonka-drone-tracker/privacy.html
- **Terms:** https://carterjensen.github.io/minnetonka-drone-tracker/terms.html

### Keywords (99 chars)
```
drone,flight,tracker,Minnetonka,police,scanner,crime,alert,safety,map,flight path,Minnesota,notify
```

### App Review Notes
Included in APP_STORE_SUBMISSION.md — explains public data source, non-affiliation, links to verify data.

### Compliance
- No "surveillance" or "tracking police" language anywhere
- Non-affiliation disclaimer in app, description, and support page
- No "Always" location permission (server-side geofencing only)
- No officer/pilot personal information displayed
- Training flights excluded from alerts
- Framed as "community awareness" not "monitoring law enforcement"

---

## Scalability Fixes Applied
1. ✅ Reads go through Supabase (not ArcGIS directly)
2. ✅ Sign in with Apple + geofences persist to Supabase
3. ✅ Push notifications in parallel batches of 50
4. ✅ Secrets removed from git (.gitignore + Secrets.swift)
5. ✅ SwiftData offline caching
6. ✅ Health check monitoring endpoint
7. ✅ Poll-flights pagination (pages of 200)
8. ✅ Error handling UI (offline banner, retry)
9. ✅ Notification deep linking to specific flights

---

## File Structure
```
MinnetonkaDroneTracker/
├── App/
│   ├── MinnetonkaDroneTrackerApp.swift
│   └── ContentView.swift
├── Core/
│   ├── Models/
│   │   ├── Flight.swift (+ ArcGIS parser)
│   │   ├── FlightPurpose.swift (icons + colors)
│   │   ├── AlertType.swift (categories + severity)
│   │   ├── Station.swift (4 stations with coordinates)
│   │   ├── Geofence.swift
│   │   ├── CachedFlight.swift (SwiftData)
│   │   └── SubscriptionTier.swift
│   ├── Services/
│   │   ├── ArcGISClient.swift (fallback only)
│   │   ├── SupabaseClient.swift
│   │   ├── FlightRepository.swift
│   │   ├── GeofenceRepository.swift
│   │   ├── AuthService.swift (Sign in with Apple)
│   │   ├── SubscriptionService.swift (StoreKit 2)
│   │   └── NotificationService.swift (APNs)
│   └── Utilities/
│       ├── PathSmoother.swift (oscillation removal + path reconstruction)
│       ├── RelativeTime.swift
│       ├── ErrorBanner.swift
│       ├── Constants.swift
│       └── Secrets.swift (gitignored)
├── Features/
│   ├── Map/ (MapTabView, MapViewModel, StationDetailView, FlightFilterSheet)
│   ├── FlightList/ (FlightListView)
│   ├── FlightDetail/ (FlightDetailView, FlightPlaybackController)
│   ├── Alerts/ (AlertsView, GeofenceEditorView)
│   ├── Subscription/ (PaywallView)
│   └── Settings/ (SettingsView)
└── Resources/ (Assets.xcassets, Info.plist, entitlements)

supabase/
├── migrations/ (001-004 SQL files)
├── functions/
│   ├── poll-flights/index.ts
│   ├── check-geofences/index.ts
│   ├── send-alerts/index.ts
│   └── health-check/index.ts
└── config.toml

app-store/
├── privacy-policy.html
├── terms-of-use.html
├── APP_STORE_SUBMISSION.md
└── store-*.png (screenshots 1284x2778)

docs/ (GitHub Pages)
├── index.html (support page + form)
├── privacy.html
└── terms.html
```

---

## Credentials & Secrets (DO NOT SHARE)
- **Supabase Project:** zkohaurrecakwbdgopam
- **Apple Team ID:** 9J8XV6HFTY
- **APNs Key ID:** ANN3K4YNA9
- **All secrets stored in Supabase Edge Function env vars**
- **iOS secrets in Secrets.swift (gitignored)**
