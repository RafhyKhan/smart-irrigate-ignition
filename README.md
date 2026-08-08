# smart-irrigate-ignition

An automated, sensor-driven lettuce irrigation system built on Ignition SCADA — combining hardware sensing, industrial control logic, safety-first automation, and a data pipeline designed for future statistical and AI-driven decision-making.

This started as an SAIT Software Development portfolio project aimed at industrial automation and SCADA roles in Calgary's oil & gas sector. It grew into a full closed-loop system: real-time soil moisture sensing, Modbus-based SCADA integration, automated actuator control with hard safety limits, and a dual-logging data pipeline (Ignition Historian + PostgreSQL) built for downstream analysis.

## Architecture

```
HW-390 Soil Moisture Sensor
        │
Arduino Nano (Modbus RTU Slave)
        │  Serial (USB / Bluetooth Classic)
Ignition SCADA (Gateway + Perspective)
        │
        ├── Gateway Timer Script (polling automation engine)
        │       ├── Moisture threshold + cooldown + daily-dose safety limits
        │       ├── Day/night watering window
        │       └── Sensor-disconnect fallback (time-based watering)
        │
        ├── Home Assistant (Docker) ──> Wemo Smart Plug ──> Water Pump
        │
        └── PostgreSQL ──> moisture_readings / watering_events
                (dual logging alongside Ignition Tag Historian)
```

## Key features

- **Live soil moisture sensing** via an HW-390 capacitive sensor, polled over Modbus RTU
- **Automated, threshold-driven watering** — waters only when moisture drops below a calibrated threshold, not on a blind timer
- **Hard safety limits, engineered after a real incident** (see below): fixed 3-second max dose, minimum cooldown between waterings, daily dose cap, day/night watering window
- **Empirically calibrated dosing** — pump flow rate measured directly (~19.8 mL/sec) rather than assumed, used to calculate watering duration from real container/soil volume
- **Dual data logging** — Ignition's own Tag Historian for in-platform dashboards, plus a parallel PostgreSQL database (`moisture_readings`, `watering_events`) built specifically for external statistical analysis
- **Perspective dashboard** — live gauge, color-coded moisture health status (dry / healthy / overwatered), pump and automation status, historical trend chart, and a watering-event log table
- **Sensor-loss resilience** — if the Modbus link drops, automation falls back to a conservative fixed schedule rather than failing silently
- **Home Assistant integration** for smart-plug actuator control, bridged from Ignition via REST calls

## The core engineering lesson: from event-driven to polling

The first automation implementation was event-driven — a tag Value Changed script that triggered watering when moisture crossed the threshold. Under real (noisy) sensor conditions, this caused a race condition: rapid, overlapping sensor updates could trigger multiple asynchronous watering cycles simultaneously, with the effect of the pump appearing to never shut off.

The fix was an architectural change, not a patch: automation was rebuilt as a **single Gateway Timer Script polling once per second**, with no event-driven triggers and no spawned async threads. This structurally eliminates the possibility of overlapping executions — the pump's 3-second maximum runtime is now a guarantee, not a best effort.

This incident, and the fix, is the single most important engineering decision in this project, and the reason the automation is trustworthy enough to run unattended.

## Tech stack

- **Hardware**: Arduino Nano, HW-390 capacitive soil moisture sensor, water pump, Wemo smart plug
- **SCADA**: Ignition (Maker Edition) — Tags, Tag Historian, Perspective, Gateway Timer Scripts
- **Protocols**: Modbus RTU (serial), REST (Home Assistant), SQL (PostgreSQL)
- **Home automation bridge**: Home Assistant (Docker, self-hosted on a UGREEN NAS)
- **Database**: PostgreSQL, managed via pgAdmin
- **Language**: Python (Jython, via Ignition's scripting engine)

## Known limitations / honest status

- Currently USB-tethered to a laptop running the Ignition Gateway; a genuine Bluetooth Classic (HC-05) link is planned but not yet stable
- No upper-bound moisture safety cutoff yet (only a lower-bound trigger) — a real gap that contributed to an overwatering incident during initial testing; planned for the next revision
- Light output (grow light) not yet independently verified against target PPFD/lux for lettuce
- No remote (outside-local-network) access yet — planned via Tailscale once Ignition is migrated to always-on NAS hosting

## Roadmap

- [ ] Add upper-bound moisture safety limit (auto-disable + alert on sustained overwatering)
- [ ] Migrate Ignition Gateway to NAS for 24/7 unattended operation independent of a personal laptop
- [ ] Swap Arduino Nano for an ESP32 (WiFi-native) to remove the local-serial-link dependency entirely
- [ ] Remote access via Tailscale
- [ ] Statistical analysis of PostgreSQL data — refine moisture thresholds and dose sizing empirically
- [ ] Growth/germination correlation analysis against a manual daily photo log
- [ ] Ignition alarm pipelines for real-time overwatering/underwatering notifications
- [ ] Local AI (RainAI) + digital-twin-gated decision layer for adaptive watering
