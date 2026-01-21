# GeoWatch

A crowdsourced atmospheric observation platform built on Nostr, inspired by iNaturalist.

## Overview

GeoWatch enables citizen scientists to document and correlate atmospheric phenomena with publicly available flight and environmental data. The platform uses decentralized Nostr protocol for censorship-resistant data sharing.

### Purpose

- Environmental monitoring and documentation
- Citizen science data collection
- Academic research on atmospheric phenomena
- Public transparency regarding aerial activities

### Key Principles

- **Public data only**: All OSINT sources (ADS-B, EPA, AERONET, ERA5) are publicly available
- **No personal tracking**: Observer identity is ephemeral by default
- **Scientific rigor**: Quality tier system distinguishes raw data from verified observations
- **Voluntary participation**: Users choose what to share; photo + location required for contributions

---

## MVP Plan

### Phase 1: Photo Submission PWA (MVP)

Minimal viable product - a Progressive Web App for voluntary photo submissions of atmospheric phenomena (clouds, contrails, exhaust plumes).

**MVP scope: Photos only.** No sensor/environmental measurements in Phase 1.

```
┌─────────────────────────────────────┐
│ 1. PROJECT SETUP                    │
│    Vite + React + TypeScript        │
│    nostr-tools, ngeohash            │
│    PWA manifest                     │
├─────────────────────────────────────┤
│ 2. CONSENT SCREEN                   │
│    First-launch warning             │
│    "You are sharing photo +         │
│    location publicly"               │
├─────────────────────────────────────┤
│ 3. CAPTURE SCREEN                   │
│    Camera (MediaDevices API)        │
│    Location (Geolocation API)       │
│    Timestamp (auto)                 │
│    Optional: observation type,      │
│    description                      │
├─────────────────────────────────────┤
│ 4. BLOSSOM UPLOAD                   │
│    Upload photo to Blossom server   │
│    Get URL + SHA256 hash            │
├─────────────────────────────────────┤
│ 5. NOSTR PUBLISH                    │
│    Generate ephemeral key           │
│    Publish kind:30383 event         │
├─────────────────────────────────────┤
│ 6. DEPLOY                           │
│    Vercel/Netlify (HTTPS required)  │
│    Test PWA install on mobile       │
└─────────────────────────────────────┘
```

### Phase 2: Sensor Submission (Post-MVP)

Add voluntary sensor reading submission using kind:4223 (per NIP PR #2163 weather station approach). Independent of photo submission - users can submit sensor data without photos and vice versa.

### Phase 3: Full PWA

- **Map view** with observation markers
- Community browsing/discovery
- Observation detail view with correlations
- Verification/identification flow
- Persistent identity option

### Phase 4: Correlation Service

- Flight path correlation (pycontrails)
- Environmental sensor correlation
- Auto-publish correlation replies

### Phase 5: Native Android

- Better camera/sensor access
- Background location
- Push notifications

---

## Event Architecture

### Event Kinds

| Kind | Type | Purpose |
|------|------|---------|
| **30383** | Addressable | Photo observations |
| **4223** | Regular | Sensor readings (NIP PR #2163) |
| **1111** | Regular | Replies: verification, correlation (NIP-22) |
| **30073** | Addressable | Community subscriptions (DIP-4) |

### Photo Observation (kind:30383)

Custom addressable event for atmospheric observations. Allows user to update/correct.

```json
{
  "kind": 30383,
  "content": "Persistent contrail with grid pattern",
  "tags": [
    ["d", "<unique-observation-id>"],
    ["I", "geo:9q8y"],
    ["I", "#geowatch"],
    ["g", "9q8yyzw8"],
    ["imeta", "url https://...", "m image/jpeg", "x <sha256>", "blurhash ...", "dim 4032x3024"],
    ["observed_at", "1705789200"],
    ["observation_type", "contrail"],
    ["t", "persistent"],
    ["l", "tier-1", "geowatch.quality"]
  ]
}
```

**Required tags:**
- `d`: Unique ID (for addressable events)
- `g`: Geohash location (7-8 char precision)
- `imeta`: Photo metadata (NIP-92)
- `observed_at`: Unix timestamp of observation

### Sensor Reading (kind:4223)

Following [NIP PR #2163](https://github.com/nostr-protocol/nips/pull/2163) weather station approach.

```json
{
  "kind": 4223,
  "tags": [
    ["I", "#geowatch"],
    ["I", "geo:9q8y"],
    ["g", "9q8yyzw8"],
    ["pm25", "45.2", "PurpleAir-PA-II"],
    ["temp", "22.5", "DHT22"],
    ["observed_at", "1705789200"]
  ],
  "content": ""
}
```

**Tag format:** `[sensor_type, value, model]`

Supported sensor types:
- Air quality: `pm25`, `pm10`, `aod`, `air_quality`
- Metals: `aluminum`, `barium`, `strontium`
- Baseline: `temp`, `humidity`, `pressure`

### Verification Reply (kind:1111)

NIP-22 comment scoped to observation.

```json
{
  "kind": 1111,
  "content": "Confirmed: persistent contrail",
  "tags": [
    ["A", "30383:<pubkey>:<d-tag>", "wss://relay"],
    ["K", "30383"],
    ["P", "<observation-author>"],
    ["a", "30383:<pubkey>:<d-tag>", "wss://relay"],
    ["k", "30383"],
    ["p", "<observation-author>"],
    ["l", "persistent-contrail", "geowatch.identification"],
    ["l", "agree", "geowatch.consensus"]
  ]
}
```

### Correlation Reply (kind:1111)

Service-generated reply linking to flight data.

```json
{
  "kind": 1111,
  "content": "Flight correlation: high confidence",
  "tags": [
    ["A", "30383:<pubkey>:<d-tag>", "wss://relay"],
    ["K", "30383"],
    ["P", "<observation-author>"],
    ["a", "30383:<pubkey>:<d-tag>", "wss://relay"],
    ["k", "30383"],
    ["p", "<observation-author>"],
    ["r", "https://opensky-network.org/api/tracks/?icao24=3c6444", "source"],
    ["flight-icao24", "3c6444"],
    ["flight-callsign", "UAL1234"],
    ["correlation-score", "0.87"],
    ["l", "flight-correlated", "geowatch.correlation"]
  ]
}
```

---

## Community Architecture

Following [DIP-4: Simple Topical Communities](https://github.com/damus-io/dips/pull/4).

### Community Types

| Type | Identifier | Example |
|------|------------|---------|
| Geographic | `I geo:<geohash>` | `["I", "geo:9q8y"]` (SF Bay) |
| Topic | `I #<hashtag>` | `["I", "#geowatch"]` |
| Relay | `I <relay-url>` | Curated/moderated feeds |

### Community Subscription (kind:30073)

```json
{
  "kind": 30073,
  "tags": [
    ["d", ""],
    ["I", "geo:9q8y"],
    ["I", "#geowatch"],
    ["I", "#contrails"]
  ],
  "content": "<optional NIP-44 encrypted private subscriptions>"
}
```

### Geohash Precision

| Chars | Size | Use |
|-------|------|-----|
| 4 | ~40km | Community subscription |
| 5 | ~5km | Community subscription |
| 6 | ~1.2km | Minimum for correlation |
| 7-8 | ~40-150m | Observation precision |

---

## Quality Tier System

| Tier | Label | Criteria | Assessor |
|------|-------|----------|----------|
| 0 | `tier-0` | Raw submission | None |
| 1 | `tier-1` | Valid photo + geohash + timestamp | Automated |
| 2 | `tier-2` | Flight correlation ≥70% | Algorithm |
| 3 | `tier-3` | 3+ community IDs agreeing | Community |
| 4 | `tier-4` | Tier 3 + sensor anomaly | Expert + data |

---

## Flight Correlation

### Data Sources

| Source | Data | Access |
|--------|------|--------|
| [OpenSky Network](https://opensky-network.org/) | ADS-B positions | Free API |
| [ADS-B Exchange](https://www.adsbexchange.com/) | Unfiltered ADS-B | RapidAPI |
| [ERA5](https://cds.climate.copernicus.eu/) | Wind at altitude | Free |

### Algorithm

```
Observation (geohash, timestamp)
         ↓
1. Expand search: ±100km, ±2hr
2. Query flights from OpenSky/ADSB-X
3. Fetch ERA5 winds (200-300 hPa)
4. Forward-advect flight paths (pycontrails DryAdvection)
5. Score spatial overlap
6. Publish correlation if score ≥ 0.70
```

Wind advection is critical: contrails drift with wind, so observation location ≠ aircraft location.

---

## Privacy Model

### Participation Tiers

| Activity | Requirements | Identity |
|----------|--------------|----------|
| Browse | None | Anonymous |
| Contribute photo | Photo + location + timestamp | Ephemeral (default) |
| Contribute sensor | Reading + location + timestamp | Ephemeral (default) |
| Build reputation | Save Nostr key | Persistent (opt-in) |

### Informed Consent (Required)

Before any submission:

```
⚠️ You are sharing:
  • Your photo (or sensor reading)
  • The location
  • The date and time

This data will be publicly visible on Nostr.

[  ] I understand
```

- First-time consent dialog with checkbox (cannot skip)
- Confirmation on every submission
- No dark patterns

### Ephemeral Identity (Default)

```javascript
// On app load
const ephemeralKey = generateSecretKey()
// Memory only - discarded on close
```

---

## Map UI

Base map implementation adapted from [Pathos](https://gitlab.com/soapbox-pub/pathos).

### Technology

| Package | Purpose |
|---------|---------|
| react-leaflet | React wrapper for Leaflet |
| leaflet | Core mapping library |
| ngeohash | Geohash encoding/decoding |

### Map Tiles

Using CARTO Positron (free, no API key required):

```typescript
// Light theme
const TILE_URL = 'https://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}{r}.png';
// Dark theme
const TILE_DARK_URL = 'https://{s}.basemaps.cartocdn.com/dark_all/{z}/{x}/{y}{r}.png';
```

### Features

- Theme-aware (light/dark mode)
- Touch/zoom support for mobile
- Geolocation with user consent
- Observation markers with thumbnails
- Geohash-based clustering at lower zoom levels
- Click marker to view observation details

### Observation Markers

```
┌─────────────────────────────────────┐
│  Individual Marker (zoom ≥ 6)       │
│  ┌───┐                              │
│  │ 📷│ ← Thumbnail from blurhash    │
│  └───┘                              │
│    │                                │
│    ▼                                │
│  Click → Observation detail view    │
├─────────────────────────────────────┤
│  Clustered Marker (zoom < 6)        │
│  ┌───┐                              │
│  │ 12│ ← Count of observations      │
│  └───┘                              │
│    │                                │
│    ▼                                │
│  Click → Zoom in / show list        │
└─────────────────────────────────────┘
```

---

## Tech Stack

### Frontend (PWA)
- React 18 + TypeScript
- Vite
- react-leaflet + leaflet (map)
- nostr-tools
- Blossom client (image upload)
- ngeohash

### Correlation Service
- Python 3.11+
- pycontrails (flight advection)
- nostr-sdk
- PostGIS
- Redis

### Data Storage
- Nostr relays (events)
- Blossom servers (images)
- No central database

---

## References

### Nostr NIPs
- [NIP-22: Comments](https://github.com/nostr-protocol/nips/blob/master/22.md) - Threading/replies
- [NIP-52: Calendar Events](https://github.com/nostr-protocol/nips/blob/master/52.md) - Geohash `g` tag
- [NIP-73: External Identifiers](https://github.com/nostr-protocol/nips/blob/master/73.md) - `I` tag for communities
- [NIP-92: Media Attachments](https://github.com/nostr-protocol/nips/blob/master/92.md) - `imeta` tag
- [NIP-B7: Blossom Media](https://github.com/nostr-protocol/nips/blob/master/B7.md) - Image storage

### Draft Proposals
- [NIP PR #2163: Weather Station Data](https://github.com/nostr-protocol/nips/pull/2163) - Sensor readings (kind:4223)
- [NIP PR #814: IoT on Nostr](https://github.com/nostr-protocol/nips/pull/814) - IoT device events
- [DIP-4: Simple Topical Communities](https://github.com/damus-io/dips/pull/4) - Community architecture

### OSINT Data Sources
- [OpenSky Network](https://opensky-network.org/) - Flight tracking
- [ADS-B Exchange](https://www.adsbexchange.com/) - Unfiltered flight data
- [US EPA AQS](https://aqs.epa.gov/) - Air quality
- [AERONET](https://aeronet.gsfc.nasa.gov/) - Aerosol optical depth
- [ERA5](https://cds.climate.copernicus.eu/) - Meteorological reanalysis

### Libraries
- [pycontrails](https://py.contrails.org/) - Contrail modeling
- [nostr-tools](https://github.com/nbd-wtf/nostr-tools) - Nostr protocol
- [Blossom](https://github.com/hzrd149/blossom) - File storage

### Datasets
- [OpenContrails](https://arxiv.org/abs/2304.02122) - Contrail detection (GOES-16)
- [GVCCS](https://arxiv.org/html/2507.18330v1) - Ground-based contrail attribution

---

## License

This project is licensed under the [GNU General Public License v3.0](https://www.gnu.org/licenses/gpl-3.0.en.html).

---

## Contributing

Contributions welcome. Please open an issue to discuss before submitting PRs.
