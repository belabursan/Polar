# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

This is a **greenfield project**. As of this writing the repository contains only documentation (`README.md` and `Doc/`) — no source code, build files, or Android project scaffolding exist yet. The full specification lives in [Doc/project_description.txt](Doc/project_description.txt); product/hardware references are the PDFs in [Doc/](Doc/). When implementation begins, this file should be updated with real build/test/lint commands.

## What is being built

**Polar** — a native Android app that builds a sailboat performance polar diagram from live NMEA 2000 telemetry. It connects to an onboard **Actisense W2K-1 Wi-Fi Gateway**, logs sailing data over time, derives "target speed" curves from the history, and renders both the historical targets and a live performance indicator on a 360° polar canvas.

## Planned technical stack

| Concern | Technology |
|---|---|
| Language | Kotlin |
| UI | Jetpack Compose, drawing via the Canvas API |
| Networking | Java sockets / Ktor — long-lived background socket thread |
| Persistence | Room (over SQLite) |

## Architecture (intended)

Three pipeline stages, matching the milestones in the project description:

1. **Ingest** — A background service holds a continuous TCP/UDP socket to the W2K-1 (Access Point mode at `192.168.4.1`, or Client mode via the boat router; data server ports `60001–60003`). The gateway is configured to emit the **N2K ASCII** output format, which delivers pre-assembled, human-readable messages — so there is **no need to reassemble fast-packet/multi-packet frames** in the app. Parse plain-text lines for the target PGNs.

2. **Filter + log** — Persist filtered samples to Room at **1 Hz**. Data-cleansing filters must drop or pause logging during unsustainable spikes (waves, tacks/jibes) or when an engine PGN appears on the bus, to keep "target" history clean. Aggregate samples into a matrix keyed by True Wind Speed (0–30 kt) × True Wind Angle (1–360°), tracking the max boat speed per cell.

3. **Visualize** — A Compose Canvas renders a radial grid (axes every 30°), concentric speed rings capped at 0–30 kt, multi-colored target-curve polylines (max historical speed per wind-speed tier), and a live marker plotting instantaneous TWA/TWS/SOW over the targets.

### Key PGNs to parse

- **PGN 128259** (Speed, Water Referenced) → Boat Speed Over Water (SOW)
- **PGN 130306** (Wind Data) → True Wind Speed (TWS) and True Wind Angle (TWA)

These three values (SOW, TWS, TWA) are the entire data model — everything downstream is derived from them.
