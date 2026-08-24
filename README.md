# Smart Irrigation system using Ignition Software



<img width="397" height="534" alt="Screenshot 2026-08-14 000949" src="https://github.com/user-attachments/assets/4040fd79-f071-4173-9889-3c823192ddfa" />
<img width="1434" height="847" alt="IgnitionDashboard13082026(2)" src="https://github.com/user-attachments/assets/a9610c7b-b261-49a4-a1e0-22f9c30251b4" />


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




# smart-irrigate-ignition

### An end-to-end, sensor-driven, closed-loop irrigation SCADA system — built from scratch, broken, debugged, and rebuilt, across hardware, industrial control software, cloud/home automation integration, and a relational data pipeline designed for future predictive modeling.

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Motivation & Goals](#motivation--goals)
3. [System Architecture](#system-architecture)
4. [Technology Stack](#technology-stack)
5. [The Build Journey — Phase by Phase](#the-build-journey--phase-by-phase)
6. [Major Incidents & Root-Cause Engineering](#major-incidents--root-cause-engineering)
7. [The Race Condition — A Deep Dive](#the-race-condition--a-deep-dive)
8. [Hardware Debugging Chronicle](#hardware-debugging-chronicle)
9. [Networking & Infrastructure Saga](#networking--infrastructure-saga)
10. [Database Engineering](#database-engineering)
11. [Automation Logic — Full Design Rationale](#automation-logic--full-design-rationale)
12. [Alerting & Notification System](#alerting--notification-system)
13. [Dashboard & Visualization](#dashboard--visualization)
14. [The Biological Side — Growing Lettuce Indoors](#the-biological-side--growing-lettuce-indoors)
15. [Skills Demonstrated](#skills-demonstrated)
16. [Known Limitations](#known-limitations)
17. [Roadmap](#roadmap)
18. [Lessons Learned](#lessons-learned)
19. [The Digital Twin: Research Foundation, Goals, and Implications](#the-digital-twin-research-foundation-goals-and-implications)
20. [On AI-Assisted Development: Prompt Engineering as a Real Skill](#on-ai-assisted-development-prompt-engineering-as-a-real-skill)
21. [The Real Skill: Troubleshooting, Not Just Coding — A Chemical Engineering Perspective](#the-real-skill-troubleshooting-not-just-coding--a-chemical-engineering-perspective)
22. [Acknowledgments](#acknowledgments)

---

## Executive Summary

This project began as a portfolio piece aimed at industrial automation and SCADA roles in Calgary's oil & gas sector, built during a Software Development diploma program. It evolved into a genuinely complete, multi-platform closed-loop control system: real-time dual-depth soil moisture sensing over Modbus RTU, industrial SCADA logic in Ignition, actuator control bridged through Home Assistant to a smart plug and water pump, dual-layer data logging (Ignition Tag Historian and a purpose-built PostgreSQL schema), a live Perspective HMI dashboard, and a script-based SMS/email alerting layer.

Along the way, the project encountered — and resolved — a genuine production-grade race condition, multiple hardware misidentification dead-ends, a NAS infrastructure failure, database authentication issues, a failed native alarm pipeline, and two full biological growing-attempt failures rooted in real environmental engineering mistakes (oversaturation, then mold from insufficient airflow). Each failure was diagnosed to its actual root cause and corrected architecturally, not patched superficially.

The system is designed from the ground up to eventually support a **digital-twin-gated AI decision layer** (a local LLM, "RainAI"), following an industrial-AI-safety pattern in which predictive actions are validated against a simulated model before being executed on real hardware.

---

## Motivation & Goals

The original design intent was explicitly layered, mirroring how real industrial automation projects are structured:

1. **Data acquisition** — physical sensors reporting real-world conditions
2. **SCADA integration** — Ignition ingesting, historizing, and alarming on that data
3. **Statistical modeling** — analyzing accumulated data for real patterns
4. **Visualization** — honest, uncertainty-aware presentation of both raw and modeled data
5. **Human insight** — a domain expert (the author) interpreting results against real theoretical constraints
6. **AI-assisted operation** — a local AI, prompt-engineered and bounded, authorized to trigger physical actuation
7. **Continuous improvement loop** — new data informing better decisions over time

This is, in essence, a human-in-the-loop industrial control system with an LLM as the last-mile actuator candidate — deliberately built bottom-up, with trustworthy sensing and safe automation established *before* any AI involvement, rather than the reverse.

---

## System Architecture

```
                         +---------------------------+
                         |  HW-390 Sensor (Shallow)   |---+
                         +---------------------------+   |
                                                          |  Analog (A0/A1)
                         +---------------------------+   |
                         |  HW-390 Sensor (Deep)      |---+
                         +---------------------------+
                                    |
                                    v
                    +-------------------------------+
                    |  Arduino Nano (SimpleModbus    |
                    |  Slave, dual-register, USB)    |
                    +-------------------------------+
                                    |  Modbus RTU (Serial, 9600 baud, 8N1)
                                    v
                    +-------------------------------+
                    |  Ignition Gateway (Maker Ed.)   |
                    |  ------------------------------ |
                    |  - OPC UA Device Connection      |
                    |  - Tags (Memory + OPC)           |
                    |  - Tag Historian                 |
                    |  - Gateway Timer Script            |
                    |    (single-threaded polling        |
                    |     automation engine, 1s tick)    |
                    |  - Perspective Dashboard            |
                    +-------------------------------+
                          |                    |
                          | REST/HTTP           | JDBC (PostgreSQL)
                          v                    v
              +-------------------+   +---------------------+
              |  Home Assistant     |   |  PostgreSQL 18        |
              |  (Docker, on NAS)   |   |  - moisture_readings   |
              |  ------------------ |   |  - watering_log        |
              |  Wemo Smart Plug    |   +---------------------+
              |        |            |
              |        v            |
              |   Water Pump        |
              +-------------------+

              +-------------------+
              |  Continuous Fan     |  (independent 5V power,
              |  (mold/humidity     |   always-on, airflow
              |   prevention)       |   management)
              +-------------------+

              +-------------------+
              |  Dual Grow Lights   |  (16h/8h cycle, mirror-
              |  + Mirror Chamber   |   assisted reflection)
              +-------------------+

              +-------------------+
              |  Gmail SMTP ->      |  Script-based text alerts,
              |  SMS Gateway        |  per-sensor, rate-limited
              |  (Telus/Public      |
              |   Mobile)           |
              +-------------------+
```

---

## Technology Stack

| Layer | Technology |
|---|---|
| Microcontroller | Arduino Nano (ATmega328P) |
| Sensors | HW-390 capacitive soil moisture (x2, shallow + deep) |
| Fieldbus Protocol | Modbus RTU (SimpleModbusSlave library) |
| SCADA Platform | Ignition 8.3.4 (Maker Edition) |
| SCADA Scripting | Jython 2.7 (Ignition's embedded Python) |
| HMI | Ignition Perspective (web-based) |
| Home Automation Bridge | Home Assistant (Docker, self-hosted) |
| Actuator | Wemo Smart Plug (Belkin) -> 5V water pump |
| NAS / Host | UGREEN NAS (UGOS), Docker via Portainer/Container Manager |
| Database | PostgreSQL 18, managed via pgAdmin |
| Notifications | Python `smtplib` -> Gmail SMTP -> carrier email-to-SMS gateway |
| Lighting | Kullsinss LED grow light strip (full spectrum, 16H timer) |
| Airflow | JZK 5V DC brushless fan (continuous operation) |
| Version Control | Git / GitHub |
| Planned AI Layer | RainAI (local LLM, Docker) |

---

## The Build Journey — Phase by Phase

### Phase 1 — Sensor & Firmware Foundations
Began with an HC-SR04 ultrasonic sensor and SG90 servo on a distance-triggered concept, using an Arduino Nano on a red I/O expansion shield. Debugged shield pin-labeling confusion, a servo-overheating scare from simultaneous USB+DC power, and DC adapter polarity/voltage settling (9V, center-positive). Wired and validated an SSD1306 OLED over I2C independently before combining subsystems.

**Pivot:** moved from distance/servo logic to soil moisture sensing (HW-390) as a more relatable, real-world SCADA use case — irrigation. Calibrated the sensor against real dry (initially ~800/733, later re-derived) and wet (~267) reference points, achieving a working 0-100% moisture readout on the OLED, validated against real lettuce-seed soil.

### Phase 2 — Modbus Integration
Attempted a switch to Modbus RTU via `SimpleModbusSlave` and hit a genuine hardware-level conflict: the Servo library and SimpleModbusSlave both required Timer1 on the ATmega328P, causing erratic sensor/display behavior. Resolved by removing the servo from the active sketch — a clean, deliberate scope reduction rather than a hacky workaround.

### Phase 3 — SCADA Onboarding
Connected the Arduino to Ignition via a Modbus RTU Device Connection over USB serial (9600, 8N1, Slave ID 1). Built the first OPC tags, enabled Tag Historian, and began learning Ignition's tag/scripting model from the ground up.

### Phase 4 — Actuation: The Smart Plug Saga
This phase alone represents one of the most extensive troubleshooting arcs in the project:

- Attempted the Wemo plug's **legacy local UPnP/XML API** directly — dead, unreachable (`:49153/setup.xml` timeout).
- Attempted **Home Assistant's HomeKit Controller integration** — blocked by HomeKit's single-controller pairing exclusivity (the device was already claimed by Apple Home).
- Discovered the integration-naming trap: Home Assistant had silently renamed "HomeKit Controller" to "HomeKit Device," a documented, widely-reported point of confusion.
- Attempted `pywemo` (direct Python library, bypassing Home Assistant) — failed to even establish a socket connection, ruling out a pairing-state issue and pointing to a deeper network-level problem.
- Diagnosed that the plug had likely never fully released its HomeKit pairing state despite appearing unpaired in Apple Home.
- **Resolution:** relocating the plug to a different physical power outlet (likely resolving a WiFi signal-strength or fresh-DHCP-negotiation issue) allowed Home Assistant's **native legacy Wemo integration** (the actual `pywemo`-backed integration, not HomeKit at all) to successfully discover and control the device.
- Home Assistant was deployed via Docker on a UGREEN NAS, configured through Portainer/Container Manager, with **bridge networking** initially (to avoid interfering with existing AdGuard DNS and Plex services), later switched to **host networking** specifically to enable mDNS/SSDP discovery for HomeKit-related attempts.

### Phase 5 — Closing the Actuation Loop
Wrote Ignition Gateway scripting to call Home Assistant's REST API (`/api/services/switch/turn_on` / `turn_off`) using a Long-Lived Access Token and Bearer authentication. Hit and resolved a genuine Ignition scripting gotcha: `system.net.httpPost()`'s third positional argument is `contentType`, not `headers` — silently swallowing the Authorization header and producing a misleading 401. Corrected by using explicit keyword arguments (`headerValues=`).

### Phase 6 — Empirical Calibration
Rather than guessing a watering duration, the pump's real flow rate was measured directly across 8 timed trials (~19.79 mL/sec, pooled average), and combined with an estimated container/soil volume to calculate a grounded ~3-second dose — a deliberately evidence-based engineering decision documented and later refined.

### Phase 7 — The First Automation (and the Incident)
Built the first automated watering logic as a `SoilMoisture` tag **Value Changed** script. Under real, noisy sensor conditions, this produced a genuine **race condition**: rapid, overlapping value-change events could each independently pass a stale `PumpBusy` check before any of them finished writing state, spawning multiple asynchronous 3-second watering cycles that overlapped and appeared to leave the pump "stuck on." *(Full analysis below.)*

The immediate consequence was a real overwatering incident — soil moisture sustained in the 72-91% range for an extended period — which, combined with an unrelated NAS power-loss event that also disrupted Home Assistant connectivity and caused the Wemo plug's DHCP-assigned IP to drift (breaking control mid-cycle), very likely killed the first batch of germinating lettuce seeds.

### Phase 8 — Architectural Redesign
Rebuilt automation from the ground up as a **single-threaded Gateway Timer Script**, polling once per second, with zero event-driven triggers and zero spawned async threads. This structurally eliminates the possibility of overlapping executions: the pump's 3-second maximum runtime becomes a mathematically guaranteed ceiling (worst-case verified via direct calculation: a 1-second poll interval guarantees no more than exactly 3.0 seconds of pump runtime), not a best-effort assumption.

### Phase 9 — Data Pipeline Engineering
Stood up a local PostgreSQL 18 instance via pgAdmin, requiring resolution of SCRAM authentication failures and a blank-password lockout (fixed via the standard `pg_hba.conf` `trust`-mode reset procedure). Built a Gateway-Timer-driven logging pipeline inserting continuous moisture readings and discrete watering events, deliberately run in parallel with Ignition's own Tag Historian for redundancy and cross-verification.

### Phase 10 — Alerting
Built a full native Ignition **Alarm Notification Pipeline** (SMTP Notification Profile via Gmail App Password, On-Call Roster, Contact records) for threshold-based text alerts. Confirmed each individual link worked in isolation — the alarm itself correctly transitioned to Active state, SMTP sending was independently verified via a raw Python `smtplib` test — but the full pipeline-to-delivery chain never successfully sent a message end-to-end, ultimately traced in part to an incorrect contact email domain (`msg.telusmobility.com` instead of the correct `msg.telus.com`) and an apparent gap between the pipeline and its Notification block. Rather than continuing to debug Ignition's alarm subsystem indefinitely, the alerting logic was **reimplemented directly inside the Gateway Timer Script** using the same proven `smtplib` code — a pragmatic engineering trade-off, explicitly favoring working software over architectural purity.

### Phase 11 — Wireless Attempt #1 and #2 (Bluetooth)
Two separate physical Bluetooth modules, both purchased and both **sold as HC-05** (genuine Bluetooth Classic / SPP), turned out on inspection to actually be **Bluetooth Low Energy (BLE)** devices exposing GATT characteristics (`FFE1`/`FFE2`) rather than a Classic serial profile. This is a documented, real-world listing-accuracy problem in this hardware category, encountered independently twice.

The first module (an AT-09/CC2541) was a genuine dead end — proper communication would have required specialized TI hardware and software (BTool, HostTest) beyond reasonable project scope.

The second module was pursued further: a Windows BLE-to-serial bridge was attempted using the open-source `ble-serial` tool alongside `com0com` (a virtual COM port driver). The BLE device was successfully scanned and its MAC address identified; `com0com`'s virtual port pair was successfully created (`CNCA2` <-> `COM9`/`BLE`); but `ble-serial` consistently failed to open the resulting virtual port with a `FileNotFoundError`, even after a full system reboot — indicating a genuine driver-activation incompatibility with the specific Windows build in use, not a configuration error. This was correctly diagnosed as a legitimate engineering dead end and abandoned in favor of continued USB-tethered operation, with a **genuine Classic-mode HC-05** (verified via product listing) ordered as the correct replacement part.

### Phase 12 — Dual-Sensor Expansion
Wired a second HW-390 sensor at a deeper soil depth, added a second Modbus holding register, and performed independent dry/wet calibration for each sensor. When recalibration produced a percentage scale incompatible with historical data (a narrower raw-value range from an insufficiently rigorous "dry" reference test), the discrepancy was resolved mathematically — deriving and verifying a linear transform between the old and new calibration scales — before ultimately choosing, for practical continuity, to reuse the original sensor's historical calibration constants rather than introduce a second, incomparable percentage scale.

### Phase 13 — Environmental Redesign (Growth Attempt #2)
After the first growing attempt's failure, built a physical grow enclosure: a cardboard box lined with strategically placed mirrors (tested in multiple configurations — flat side walls, then an angled "reflector hood" above the light source, approximating commercial grow-tent reflector geometry), dual grow lights, a waterproof plastic tray base, and ventilation gaps. This second attempt also stalled — no germination after several days, and visible mold/fungal growth was discovered near the sensor probe, diagnosed as a consequence of the fully enclosed, low-airflow, highly reflective chamber trapping humidity.

**Corrective redesign:** fresh (non-contaminated) soil, a continuously-running 5V DC fan for airflow (deliberately powered independently of the intermittently-connected Arduino, to guarantee 24/7 operation), and a shallower sensor repositioning to actually match the seed germination depth (previously mismatched — sensor at 2", seeds at 1/8"-1/4"). Post-fan monitoring showed the expected physical response: surface moisture reading dropped, the shallow/deep moisture gradient narrowed as expected from active airflow, and soil texture shifted from saturated to an appropriately damp "wrung-sponge" consistency.

### Phase 14 — Automation Finalization
Iteratively refined automation logic based on direct empirical observation rather than theoretical assumption — including recognizing that a fixed calendar-based watering schedule (once daily, matching an informal control-group comparison against a friend's manually-watered, sun-grown lettuce) combined with a moisture floor was more practical than a purely reactive threshold system, while retaining an independent manual **Force Automation** override capable of bypassing the automation-disable safety toggle for on-demand, fully-logged watering events. Rebuilt the PostgreSQL schema from scratch (after exporting historical data to CSV) into two clean, purpose-built tables with explicit date/time columns and per-event trigger-source notes, resolving a PostgreSQL/JDBC type-casting gotcha (`character varying` vs. `date`/`time` parameter binding) along the way.

---

## Major Incidents & Root-Cause Engineering

| # | Incident | Root Cause | Resolution |
|---|---|---|---|
| 1 | Servo + Modbus Timer1 conflict | Shared hardware timer resource | Removed servo from active sketch |
| 2 | Wemo plug unreachable | HomeKit-exclusive pairing state | Migrated to Home Assistant's native legacy Wemo integration |
| 3 | **Pump stuck on (race condition)** | Overlapping async triggers from noisy Value-Changed events | Rebuilt as single-threaded Gateway Timer polling architecture |
| 4 | **Overwatering incident** | Race condition + NAS power loss + DHCP IP drift | Architectural fix (above) + planned upper-bound safety monitoring |
| 5 | PostgreSQL auth failures | SCRAM misconfiguration, blank password | Standard `pg_hba.conf` trust-mode reset |
| 6 | Alarm pipeline silent failure | Wrong contact email domain + pipeline/block disconnect | Reimplemented alerting directly in script |
| 7 | Two "HC-05" modules were actually BLE | Hardware listing inaccuracy (documented, recurring issue) | Diagnosed via GATT characteristic inspection; ordered verified Classic module |
| 8 | BLE-to-serial bridge failure | Windows driver activation incompatibility (`com0com`) | Correctly identified as a genuine dead end; reverted to USB |
| 9 | Recalibration produced incomparable data | Insufficiently rigorous dry-point reference test | Solved via linear-transform math; reused original calibration for continuity |
| 10 | No germination, mold discovered | Enclosed, low-airflow, high-humidity chamber | Continuous independent-power fan; fresh soil; corrected sensor depth |
| 11 | Watering log table empty after query rebuild | Named Query binding / PostgreSQL date-type casting mismatch | Explicit `::date`/`::time` casts in parameterized SQL |

---

## The Race Condition — A Deep Dive

This is worth documenting in detail, as it is the project's single most significant piece of engineering judgment.

**Original design:** a `Tag Event Script` on `SoilMoisture`'s `Value Changed` event checked a `PumpBusy` boolean tag; if `false`, it set `PumpBusy = true`, commanded the pump on, and spawned an asynchronous 3-second delayed shutoff.

**The failure mode:** the Arduino's raw sensor readings were not perfectly stable — minor electrical noise produced frequent, small value fluctuations, each of which independently fired the `Value Changed` event. Because the "check `PumpBusy`, then set `PumpBusy`" sequence was not atomic, two events arriving in close succession could **both** read `PumpBusy` as `false` before either had finished writing `true` — a textbook time-of-check-to-time-of-use (TOCTOU) race condition. Each spawned its own independent asynchronous timer thread. With multiple overlapping cycles each independently trying to turn the pump on and off on staggered schedules, the pump appeared, externally, to simply never turn off.

**Why the fix works structurally, not just empirically:** the replacement Gateway Timer Script runs on a single execution thread, on a fixed one-second interval, entirely decoupled from how often or how erratically the underlying sensor value changes. There is no `PumpBusy`-style check-then-set race, because there is no possibility of two overlapping executions — the automation logic only ever runs from one place, sequentially. The maximum pump runtime is not a *statistical likelihood* of staying near 3 seconds; it is a **provable upper bound**, verified via direct calculation: with a 1-second poll interval, worst-case pump runtime is exactly 3.0 seconds, full stop — because the poll interval evenly divides the target duration.

---

## Hardware Debugging Chronicle

- Shield pin-labeling front/back confusion (HC-SR04)
- Dual USB+DC power conflict causing servo overheating
- DC adapter polarity/voltage determination (9V, center-positive)
- I2C OLED address verification (0x3C vs 0x3D)
- Timer1 resource conflict (Servo vs. Modbus)
- Soil sensor depth/seed-depth mismatch (2" sensor vs. 1/8"-1/4" seeds)
- Two independent BLE-mislabeled-as-Classic-Bluetooth hardware failures
- USB upload failures traced to HC-05 wiring sharing the same TX/RX lines as the USB-to-serial chip (resolved by disconnecting Bluetooth wiring during firmware uploads)
- A flaky USB cable initially misdiagnosed as a board failure, correctly re-diagnosed via systematic elimination (cable swap, port swap, driver update)
- Voltage-divider design and construction (1kOhm + 2x1kOhm-in-series, breadboard-based) for safe 5V->3.3V logic-level shifting on the HC-05 RXD line
- Dual-sensor independent calibration and a full linear-algebra reconciliation between two calibration epochs

---

## Networking & Infrastructure Saga

- UGREEN NAS Docker deployment (Portainer / native Container Manager)
- Bridge vs. host Docker networking trade-offs (mDNS/SSDP discovery vs. avoiding interference with AdGuard/Plex)
- SSH access troubleshooting (host-key mismatch warnings, correct-vs-router IP confusion, Docker daemon group permissions)
- PostgreSQL SCRAM authentication recovery via `pg_hba.conf`
- A full NAS abrupt-power-loss recovery (Home Assistant SQLite database unclean-shutdown recovery, Docker container restart)
- DHCP IP-address drift on both the Wemo plug (observed across three different IPs in one session) and, separately, verification steps for the NAS itself — leading directly to a stated future goal of DHCP reservations for both devices
- Gmail SMTP App Password configuration (distinct from standard account password, requiring 2FA)
- Carrier email-to-SMS gateway research and verification (Public Mobile routes through Telus's `msg.telus.com` domain, confirmed via direct empirical test)
- Windows multi-Python-interpreter package installation confusion (`pip install` targeting a different interpreter than the one executing the script) — resolved via explicit full-path interpreter invocation

---

## Database Engineering

**Schema evolution:** began with a single combined `moisture_readings` and `watering_events` table pair, iterated through a `moisture_readings_dual` intermediate design, and was ultimately purged and rebuilt (after CSV export for data preservation) into a clean, minimal two-table schema:

```sql
CREATE TABLE moisture_readings (
    id SERIAL PRIMARY KEY,
    reading_date DATE,
    reading_time TIME,
    moisture_shallow NUMERIC,
    moisture_deep NUMERIC,
    note TEXT
);

CREATE TABLE watering_log (
    id SERIAL PRIMARY KEY,
    event_date DATE,
    event_time TIME,
    duration_sec INTEGER,
    estimated_ml NUMERIC,
    moisture_shallow_before NUMERIC,
    moisture_deep_before NUMERIC,
    moisture_shallow_after NUMERIC,
    moisture_deep_after NUMERIC,
    note TEXT
);
```

**Notable engineering decisions:**
- Separate `DATE`/`TIME` columns (rather than a single `TIMESTAMP`) chosen deliberately for simpler downstream Excel filtering/sorting
- A deferred-write pattern for post-dose moisture: each watering event schedules a follow-up check exactly one hour later, at which point the same row is updated in place with `moisture_shallow_after`/`moisture_deep_after` — giving a genuine before/after causal snapshot per dose, not just a point-in-time reading
- A `note` column carrying an explicit trigger-source label (`"Scheduled Automation (12:00pm)"` vs. `"Force Automation"`), enabling later analysis to distinguish automated from manually-triggered events
- A `0`-as-sentinel convention for sensor-unavailable readings (rather than `NULL`), chosen deliberately for straightforward filtering (`WHERE moisture_shallow_before > 0`) given the system's intermittent laptop connectivity
- Resolution of a PostgreSQL/JDBC parameter-binding gotcha: `system.db.runPrepUpdate()` passes string parameters as `character varying` by default; inserting into `DATE`/`TIME`-typed columns required explicit `::date`/`::time` casts in the parameterized SQL text itself

---

## Automation Logic — Full Design Rationale

The final Gateway Timer Script architecture reflects an explicit design philosophy: **safety guarantees must be structural, not statistical.**

- **Single execution thread, fixed poll interval** — eliminates race conditions by construction
- **Hard-capped dose duration** (3 seconds) — mathematically bounded, not best-effort
- **Independent Force/Scheduled trigger paths** — `AutomationDisabled` gates only the scheduled path, allowing manual intervention (fully logged, identical safety guarantees) even while the automatic schedule is paused
- **Sensor-quality-aware logging** — every database write checks tag quality before use, substituting a sentinel value rather than silently logging garbage or crashing
- **Deferred causal verification** — the one-hour-later moisture check is itself implemented via the same single-threaded polling pattern (a `PendingCheckTime` tag compared against `now` on every tick), not a separate timer thread, preserving the same race-condition-free guarantee

This progression — from a plausible-looking but subtly unsafe event-driven design, through a real incident, to a provably safe polling architecture — mirrors exactly the kind of judgment industrial control engineers are expected to develop, typically over a much longer timeline than this project's compressed build cycle.

---

## Alerting & Notification System

Two independent alerting layers were built:

1. **Native Ignition Alarm Pipeline** (SMTP Notification Profile, On-Call Roster, Contact records) — built to completion, individually verified at every stage, but never successfully closed the full delivery loop. Retained in the project as a demonstration of Ignition's native alarming capability and as a documented "known gap" for future resolution, rather than deleted.
2. **Direct script-based SMTP alerting** — `smtplib`-based, triggered from `Value Changed` events on both moisture sensors independently, with a per-sensor rate-limiting cooldown (tuned iteratively from 30 minutes down to 5 minutes to 1 hour based on observed alert volume and Gmail's sending limits, empirically confirmed to be well within personal-account thresholds).

---

## Dashboard & Visualization

Built entirely in Ignition Perspective:

- Live moisture gauge (0-100%)
- Color-coded five-zone health status label (dry / getting low / healthy / getting wet / overwatered), using nested Expression-language conditionals
- Pump and automation status indicators with dynamic background-color bindings
- A dual-line Time Series Chart (shallow vs. deep moisture, distinct colors) — implemented via a custom Script Transform reshaping Tag History (and later, direct PostgreSQL Named Query) datasets into the multi-series structure Perspective's chart component expects
- A scrollable Watering Log table, bound directly to PostgreSQL via a Named Query
- Manual override controls (Force Automation trigger button, Automation Disabled toggle)
- Design discussions around a fully custom SVG/Coordinate-Container "live plant diagram" — a water-level-style fill gauge, positioned sensor markers, and pump-activity iconography — representing an explored but not yet fully implemented advanced Perspective visualization technique

---

## The Biological Side — Growing Lettuce Indoors

This project treats plant biology with the same engineering rigor as the software/hardware stack, grounded in real horticultural research rather than assumption:

- Germination timeline expectations (7-14 days to emergence) established from research rather than impatience-driven troubleshooting
- Correct seed-sowing depth (1/8"-1/4") identified and reconciled against original sensor placement (2" — a genuine design flaw, corrected)
- The "wrung-out sponge" moisture standard adopted as the practical target, replacing an initially arbitrary percentage-based threshold
- Explicit recognition that light and water are not independent variables — water without adequate light is not merely wasted but actively counterproductive, informing the decision to prioritize verifying grow-light output (via lux measurement) as highly as moisture control
- A deliberate day/night watering lockout, informed by the reasoning that irrigation during a plant's dark cycle serves no physiological purpose and only increases oversaturation risk
- Two full growing-attempt failures, each diagnosed to a specific, correctable root cause (oversaturation; then mold from insufficient airflow) rather than treated as unexplained bad luck
- An informal but methodologically honest comparison against a friend's manually-watered, naturally-sunlit control, explicitly acknowledging the confound (real sunlight vs. artificial grow light) rather than overclaiming a clean experimental comparison

---

## Skills Demonstrated

**Ignition/SCADA:** Tags (Memory, OPC), Tag Historian (sample-mode/deadband tuning), Gateway Timer Scripts, Tag Event Scripts (built, then deliberately deprecated for sound architectural reasons), Perspective (Views, Gauge, Label, Toggle, Button, Table, Time Series Chart, Coordinate Containers, multiple binding types including Tag, Expression, Tag History in Wide/Tall/AsStored formats, Query bindings, Script Transforms), Named Queries, Database Connections (JDBC/PostgreSQL), Modbus RTU device configuration, Alarm Notification Pipelines (Profiles, Rosters, Contacts), `system.db`/`system.net`/`system.tag`/`system.date`/`system.dataset` scripting APIs.

**Embedded/Hardware:** Arduino C++, analog sensor calibration, Modbus RTU slave implementation, voltage-divider circuit design, breadboard prototyping, timer-resource conflict debugging, BLE vs. Bluetooth Classic protocol distinction, serial communication debugging.

**Software Engineering:** race-condition diagnosis and structural (not superficial) remediation, single-threaded-vs-async architectural trade-off reasoning, defensive scripting (quality checks, sentinel values, try/except boundaries), REST API integration (Home Assistant), SMTP email automation.

**Data Engineering:** PostgreSQL schema design and iteration, SQL (DDL, DML, parameterized queries, type casting), JDBC parameter-binding gotchas, data export/preservation discipline before destructive schema changes.

**Systems/Infrastructure:** Docker deployment (Portainer, Container Manager), NAS administration, SSH, DNS/DHCP troubleshooting, network diagnostics (`ping`, `ipconfig`, connection-exception root-causing), Windows driver-level troubleshooting (`com0com`), multi-Python-interpreter environment management.

**Domain Knowledge:** industrial control safety patterns (structural vs. statistical guarantees), horticultural science applied to an engineered system, empirical calibration methodology (measured flow rates, linear-transform calibration reconciliation).

---

## Known Limitations

- Currently USB-tethered to a personal laptop; genuine wireless (verified Classic HC-05) pending
- No upper-bound moisture safety monitor yet implemented (only a lower-bound watering trigger) — identified as the single highest-priority safety gap given the overwatering incident history
- Native Ignition Alarm Pipeline delivery chain remains unresolved
- No remote (outside-home-network) dashboard/control access yet
- Temperature/humidity and light-intensity sensing not yet implemented — currently inferred qualitatively rather than measured
- No confirmed successful germination as of the current build (two prior attempts failed; a third is in progress under corrected conditions)

---

## Roadmap

- [ ] Upper-bound moisture safety monitor with independent alerting
- [ ] Genuine Classic-mode HC-05 wireless integration
- [ ] ESP32 migration (native WiFi, MQTT-based communication, eliminating the serial/Bluetooth dependency entirely)
- [ ] Ignition Gateway migration to always-on NAS hosting
- [ ] Tailscale-based remote access
- [ ] DHT temperature/humidity sensor integration
- [ ] Light intensity (lux/PPFD) sensor integration
- [ ] Daily photo-log-based growth quantification
- [ ] UDT-based tag templating for multi-sensor scalability
- [ ] Transaction Groups as a native-Ignition alternative to hand-rolled SQL logging
- [ ] Statistical analysis of accumulated PostgreSQL data (moisture/watering/outcome correlation)
- [ ] A genuine digital twin — a regression model trained on the project's own historical data, predicting outcomes before actions execute
- [ ] RainAI integration, gated by the digital twin's predictions, following the industrial AI-safety pattern of simulation-before-action

---

## Lessons Learned

1. **Structural safety beats statistical safety.** A system that is *usually* fine under normal conditions is not the same as a system that is *guaranteed* fine under all conditions — the race condition taught this distinction concretely, not abstractly.
2. **Hardware listings lie, sometimes twice in a row.** Two separately-purchased "HC-05" modules were both actually BLE devices. Verifying actual protocol behavior (GATT characteristics vs. Classic SPP) mattered more than trusting product titles.
3. **Biological systems punish impatience and reward measurement.** Both growing failures were caused by controllable environmental factors, correctly diagnosed only once actual physical checks (soil texture, mold inspection, temperature) were treated as seriously as sensor numbers.
4. **Know when to stop.** The BLE-bridge troubleshooting session, the native Alarm Pipeline debugging, and the SSH password saga each represent moments where continued effort had genuinely diminishing returns — recognizing and stepping back from a dead end is itself an engineering skill, not a failure.
5. **Empirical grounding beats assumption, every time.** The pump's flow rate was measured, not guessed. The watering duration was calculated, not chosen arbitrarily. The moisture thresholds were tuned against physical soil-feel checks, not held rigidly to a first-guess number. This discipline is the connective thread across the entire project.

---

## On AI-Assisted Development: Prompt Engineering as a Real Skill

This project was built in close collaboration with Claude (Anthropic), used extensively across nearly every layer of the stack — Arduino firmware, Ignition scripting, SQL, Docker/NAS configuration, and this documentation itself. It's worth being explicit and honest about what that collaboration actually looked like, because the *way* AI was used here is arguably as demonstrative of real engineering skill as the code itself.

**What AI-assisted development is not, in this project:** it was never "describe the finished feature, receive working code, ship it." Nearly every script above went through multiple failed iterations, live debugging against real hardware, and correction based on actual runtime behavior that no amount of prompting alone could have predicted. The race condition, the JDBC type-casting bug, the BLE-vs-Classic hardware misidentification, the `httpPost` argument-order gotcha — none of these were things Claude got right on the first try, and none of them were things that could be fixed by re-prompting alone. They required the author to actually run the code, read the real error messages, form a hypothesis, and direct the next iteration — the same loop a solo developer runs internally, just with a collaborator in it.

**Where prompt engineering actually mattered:**
- **Precise problem framing over vague requests.** Pasting a raw stack trace or an exact error string (`SQLException: column "reading_date" is of type date but expression is of type character varying`) produced immediately actionable fixes; vague descriptions ("it's not working") did not, and were explicitly avoided once this pattern became clear.
- **Providing ground truth Claude cannot access.** Claude cannot see a physical Bluetooth LED blink pattern, feel soil moisture, smell mold, or read a multimeter. Every hardware diagnosis in this project depended on the author supplying that sensory ground truth back into the conversation, which Claude then reasoned over — a genuinely bidirectional process, not a one-way code request.
- **Pushing back on confident-sounding answers.** Several points in this project's history involved directly challenging an AI-suggested conclusion (e.g., questioning an "upper safety limit" design that turned out to be logically redundant with the existing lower threshold, or pushing back on an over-broad "the whole batch might be dead" inference from one localized observation of mold). In each case, the pushback led to a better, more precisely-scoped answer — a reminder that AI output, like any other engineering input, should be verified and reasoned about, not accepted at face value.
- **Knowing when to redirect versus when to accept a stopping point.** The BLE-bridge troubleshooting session is the clearest example: after `com0com` and `ble-serial` both failed even after a full reboot, continuing to prompt for another fix would have been the wrong move — recognizing a genuine dead end (with AI's help in framing it as one) and reverting to a known-working configuration was the more disciplined engineering call.
- **Using AI for what it's actually good at.** Boilerplate generation (repetitive SQL, repeated tag-script patterns), calculation double-checking (flow rate math, calibration linear transforms, worst-case timing analysis — deliberately run through an actual calculator/interpreter rather than trusted from memory), and rapid iteration on Ignition's less-documented scripting quirks were all areas where AI assistance genuinely accelerated the work without substituting for understanding it.

**The actual estimate of prompt volume:** this project spans several hundred individual prompts across multiple extended sessions, covering firmware, SCADA configuration, database design, infrastructure troubleshooting, horticultural research, and documentation — a volume that reflects the project's genuine end-to-end scope rather than any single "one-shot" request. The throughline across all of them is the same one that runs through the rest of this README: nothing here was accepted as correct because Claude said so; everything was accepted as correct because it was tested against real hardware, real data, or real physical observation.

---

## The Real Skill: Troubleshooting, Not Just Coding — A Chemical Engineering Perspective

This project's author began in Chemical Engineering before transitioning into Software Development — a background that turned out to be directly, unexpectedly relevant, because process engineering and software/controls engineering share the same underlying discipline: **systems with feedback loops fail in non-obvious ways, and the job is root-cause diagnosis, not pattern-matching to a textbook answer.**

Writing code — or prompting an AI to write it — is a comparatively small fraction of what this project actually required. What actually consumed the overwhelming majority of the effort, and what most directly reflects the skills a modern developer needs, was troubleshooting across layers that don't share a common language: electrical (voltage dividers, timer conflicts), firmware (serial protocol conflicts), network (DHCP drift, DNS, Docker bridging), software architecture (race conditions), database internals (JDBC type coercion), and — least "software" of all — biology and thermodynamics (evaporation rates, mold growth conditions, germination physiology).

**A chemical-engineering lens on several of this project's specific problems:**

- **The race condition is, structurally, a mixing/residence-time problem.** Multiple overlapping "batches" (watering cycles) entering a shared system (the pump state) without proper sequencing is conceptually identical to short-circuiting flow in an improperly baffled reactor — two streams that should be processed in series instead interfere with each other because there's no enforced ordering. The fix (a single-threaded polling loop) is the software equivalent of adding a proper plug-flow constraint: force strict sequential processing so nothing overlaps.
- **The oversaturation incident is a mass-balance problem.** Water was entering the system (via the pump) faster, and more erratically, than it could leave (via evaporation, drainage, and plant uptake) — a classic accumulation fault, the same category of problem as a tank overflowing because inflow control failed independently of outflow capacity. The eventual fix — moving from a purely reactive lower-threshold trigger toward also reasoning about *rate of change* and *upper bounds* — mirrors how a real process control system would add both low-level and high-level interlocks, not just one.
- **The mold/airflow failure is a heat-and-mass-transfer problem.** A fully enclosed, reflective chamber increased light-driven heat retention while simultaneously reducing convective mass transfer (evaporation) at the soil-air boundary — the exact mechanism by which humidity accumulates in an insufficiently ventilated real chemical process vessel. The fix (forced convection via a continuously-running fan) is literally the same intervention a process engineer would specify to break a stagnant boundary layer and restore evaporative mass transfer.
- **The calibration reconciliation problem is a units/reference-frame conversion problem.** Two sensors reporting on different raw scales needed to be reconciled onto a common basis before their readings could be meaningfully compared — directly analogous to converting between two different pressure or concentration reference standards in a lab, requiring the same disciplined "solve for the transform, verify it maps both known points correctly" approach used in the linear-algebra reconciliation documented above.
- **Empirical measurement over assumed specification, throughout.** The decision to physically time the pump's flow rate across 8 trials rather than trust a datasheet number, and to physically test soil texture and germination status rather than rely purely on a sensor's calibrated percentage, reflects a core chemical/process engineering instinct: **instrumentation is a model of reality, not reality itself, and must be periodically validated against direct physical observation.** This instinct directly prevented at least one wrong conclusion in this project — the shallow sensor's flat 0% reading, which resolved once physical soil contact (not sensor logic) was corrected.

**Why this matters for a hiring manager reading this repository:** the specific tools here — Ignition, PostgreSQL, Arduino, Home Assistant — are all learnable in weeks. What is not quickly learnable, and what this project's incident log is meant to demonstrate, is the underlying diagnostic discipline: forming a hypothesis, designing the cheapest test that would falsify it, reading the actual evidence (an error string, a raw sensor value, a soil texture, a mold patch) rather than the expected evidence, and only then committing to a fix. That discipline transfers directly from a chemical engineering process-troubleshooting background into software/controls engineering, and this project is, among other things, a demonstration of that transfer in action.

---

## The Digital Twin: Research Foundation, Goals, and Implications

Before any AI-actuation work began, the project's design was directly informed by industrial research into how AI performs when it is given control authority over physical systems. The specific, motivating finding: **AI systems that operate with the assistance of a digital twin — a live, predictive simulation of the system they are controlling — perform measurably better and more safely than AI systems that act directly on raw sensor input and instructions alone.** This became the guiding design constraint for every AI-adjacent decision made in this project, well before any AI-driven actuation was actually implemented.

**Why this mattered enough to shape the architecture early:** the naive design for "let an AI water the plant" would be to hand a language model the current moisture reading and a natural-language instruction, and let it decide whether to trigger the pump. This project deliberately did not build that. The research is clear that this pattern — an AI reasoning directly from a live measurement to a real-world action, with no intermediate check — is exactly the pattern responsible for the more serious documented AI-control failures in industrial contexts, because the AI has no way to distinguish a real signal from a sensor artifact, a genuine trend from noise, or a locally reasonable action from one that compounds into a systemic failure over time.

**What a digital twin actually adds, and why it is not just "more code":** a digital twin is a model — in this project's case, ultimately a regression model trained on the system's own historical data — that predicts *what will happen* if a proposed action is taken, before that action is executed. The AI does not act on the world directly; it proposes an action, the digital twin simulates the likely outcome, and only if that predicted outcome falls within safe, expected bounds does the action proceed. This is structurally the same principle as the project's core software-safety lesson (see: *The Race Condition — A Deep Dive*) applied one layer up the stack — just as the pump's automation logic was rebuilt to make safety a structural guarantee rather than a statistical likelihood, the eventual AI layer is designed so that safety is enforced by a prediction-and-check gate, not by trusting the AI's judgment alone.

**Concrete implications this had for the project, already realized:**
- **Data pipeline built for prediction, not just logging.** The PostgreSQL schema's deliberate before/after-dose moisture capture (`moisture_shallow_before`, `moisture_shallow_after`, captured exactly one hour apart) exists specifically to eventually support training a twin: a genuine paired input/output dataset ("given this starting condition and this action, this was the measured outcome"), not just a raw time series.
- **A working precedent for the twin already exists, informally.** The empirical flow-rate measurement and the linear-transform calibration reconciliation are, in miniature, exactly what a digital twin does: build a mathematical model from measured data, and use that model to predict/convert rather than trusting an assumed value.
- **The upper-bound safety gap, identified as the project's top unresolved risk, is precisely the kind of check a digital twin would formalize.** Right now the system only knows to act ("water if below threshold"); it does not yet reason about trend or predict an outcome before acting. A twin-based design would replace "water because moisture is low right now" with "water because the model predicts moisture will remain low for the next N hours without intervention" — a materially safer decision rule.

**Goals for the digital twin phase, concretely:**
1. Train a simple regression model on the project's own accumulated moisture/watering/outcome data — predicting moisture decline rate as a function of recent conditions (light cycle, time since last dose, ambient temperature once that sensor is added).
2. Use that model to predict, before any dose fires, what moisture is expected to do over the following hours — both with and without intervention.
3. Gate any AI-proposed action (from RainAI) behind this prediction: the AI may *suggest* a change (e.g., "lower the threshold, growth looks slow"), but the digital twin's prediction is what determines whether that suggestion is safe to actually execute, not the AI's stated confidence in its own recommendation.
4. Extend the same pattern to the light and airflow subsystems as sensors for those variables come online, so the twin eventually models the whole environmental system the plant experiences, not moisture in isolation.

**The honest, larger point this research shapes:** the project's AI ambitions are deliberately the *last* phase, not the first, and this is not incidental caution — it is a direct, considered response to documented evidence about how AI-controlled physical systems actually fail. Every phase preceding it (reliable sensing, structurally-safe automation, a clean and causally-structured dataset) exists specifically to make the eventual digital twin trainable and trustworthy, rather than being built in parallel with, or as an afterthought to, the AI layer itself.

---



Built as an independent portfolio project during a Software Development diploma program, following a Chemical Engineering background, targeting industrial automation and SCADA roles in Calgary's oil & gas sector. Special engineering credit to the process of debugging a live, real-world race condition affecting an actual living plant — a constraint that made every safety decision genuinely consequential rather than theoretical.

---

*This README documents the full engineering journey, including failures, as a deliberate choice — the diagnostic process behind each incident is considered at least as valuable, for demonstrating real engineering judgment, as the final working state of the system itself.*


