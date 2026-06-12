# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

This is a **greenfield project**. As of this writing the repository contains only documentation — no source code, build files, or Android project scaffolding exist yet. The full, authoritative specification lives in [Doc/project_description.txt](Doc/project_description.txt) (read it first); product/hardware references are the PDFs in [Doc/](Doc/). When implementation begins, update this file with the real build/test/lint commands.

## What is being built

**Polar** — a native Android app that builds a sailboat performance polar diagram from live NMEA 2000 (N2K) telemetry. It connects to an onboard **Actisense W2K-1 Wi-Fi Gateway**, logs sailing data over time, derives rolling "target speed" curves from the history, and renders both the targets and a live performance indicator on a 360° polar canvas. It also tracks GPS position so the last 3 runs can be reviewed as a speed-colored trace on a nautical map.

The W2K-1 is configured to emit the **N2K ASCII** plain-text format, so the app reads pre-assembled, human-readable messages — there is **no need to reassemble fast-packet/multi-packet frames**.

## Architecture — the one thing to understand first

The project is an **Android multi-module Gradle** build whose governing rule is **dependency inversion for portability**: all contracts (interfaces + domain models) live in the pure-Kotlin **`:core`** module, and *every other module depends only on `:core`* — never on each other. A thin **`:app`** module wires concrete implementations together at runtime via a factory that reads settings. This is deliberate: it lets modules (e.g. communication) be lifted into other projects, and lets implementations be swapped (Wi-Fi → Bluetooth, NMEA 2000 → another protocol) without touching the rest of the system. **Preserve this rule** — if you find yourself adding a module-to-module dependency, the contract belongs in `:core` instead.

### Modules (12)

| Module | Role | Platform |
|---|---|---|
| `:app` | Entry point, DI (Hilt), foreground Service host, transport/protocol factory. Glue only. | Android |
| `:core` | Contracts hub: all domain models + all interfaces. Depends on nothing. | Pure Kotlin |
| `:comm:transport` | Raw transport, emits framed `Flow<String>`. Wi-Fi now, Bluetooth later. | Pure Kotlin |
| `:comm:nmea` | N2K ASCII parser → `TelemetrySample` + `PositionFix`. Takes a `Transport` by injection. | Pure Kotlin |
| `:logic:timing` | 1 Hz throttling of the stream. | Pure Kotlin |
| `:logic:filtering` | Cleansing: drop spikes, tacks/jibes, engine-PGN-active periods. | Pure Kotlin |
| `:logic:calculation` | TWS×TWA matrix, max-SOW tracking, target-curve derivation. | Pure Kotlin |
| `:data` | Room persistence; implements `SampleRepository` + `TrackRepository`. | Android |
| `:ui` | Compose: polar canvas + osmdroid map (OSM base + OpenSeaMap overlay). | Android |
| `:logging` | Rolling diagnostic logs (size from `:config`), gzip, email export. Implements `Logger`. | Android |
| `:config` | DataStore settings, reactive `Flow<Settings>`. Implements `SettingsRepository`. | Android |
| `:tracking` | Groups `PositionFix` into runs (last 3), speed→color, N2K-preferred/phone-fallback merge. | Pure Kotlin |

The split into **`:comm:transport`** (generic, knows nothing about NMEA) and **`:comm:nmea`** (the parser) is intentional — it keeps the transport reusable. Keep parsing out of the transport module.

### Data flow

- **Performance:** `:comm:transport` (lines) → `:comm:nmea` (`TelemetrySample`) → `:logic:timing` (1 Hz) → `:logic:filtering` → `:data` (persist) + `:logic:calculation` (matrix/curves) → `:ui` (live marker + target curves).
- **Tracking:** `PositionSource` (N2K position preferred, phone GPS fallback) → `:tracking` (runs, speed→color) → `:data` → `:ui` (speed-colored trace on the map).

`:app` reads `Flow<Settings>` and a factory selects the matching `Transport` (comm type) and parser (protocol). Adding a protocol/transport/position source later = one new impl module + one enum value + one factory branch.

### Key PGNs

| PGN | Data | Used for |
|---|---|---|
| 128259 | Speed, Water Referenced → **SOW** | polar performance |
| 130306 | Wind Data → **TWS, TWA** | polar performance |
| 129025 | Position, Rapid Update → **lat/lon** | tracking |
| 129026 | COG & SOG, Rapid Update → **COG, SOG** | tracking |
| 127250 | Vessel Heading → **true heading** | tracking |

`TelemetrySample(sow, tws, twa, timestamp)` and `PositionFix(lat, lon, timestamp, sog, cog, heading)` are the whole data model — everything downstream derives from them.

## Tech stack

| Concern | Technology |
|---|---|
| Language / async | Kotlin; Coroutines + Flow (the spine; keeps pure modules Android-free) |
| UI | Jetpack Compose (Canvas API for the polar plot) |
| Networking | Java sockets / Ktor — long-lived socket in an Android foreground Service |
| Persistence | Room (SQLite) |
| Settings | Jetpack DataStore |
| Maps | osmdroid — OSM base + OpenSeaMap seamark overlay, offline-tile capable |
| DI | Hilt (in `:app` only; pure modules expose plain constructors) |

Build defaults: minSdk 26, latest compileSdk, Kotlin DSL build scripts, `libs.versions.toml` version catalog.

## Git workflow

- `master` — releases only.
- `develop` — integration branch for ongoing work.
- `feature/*` — branch off `develop` per feature, merge back into `develop`.
