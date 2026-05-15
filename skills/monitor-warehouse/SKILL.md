---
name: monitor-warehouse
description: |
  Help warehouse managers, ops supervisors, and shift leads understand what is happening at their site from connected sensors, and surface the key numbers on screens.

  **Perfect for:**
  - Checking whether a sensor is alive or has gone offline
  - Reading the current value of something (cold storage temp, dock door state, conveyor status)
  - Rolling up recent activity (shipments today, doors opened, scans per hour)
  - Diagnosing why a screen is not showing data from a sensor
  - Picking what to surface on a breakroom or lobby screen

  **Not ideal for:**
  - Historical analysis beyond event-log retention
  - Cross-device joins or SQL-style aggregates (events live per-device)
  - Sub-second alerting (device data is poll-based, 5-10s latency)
  - Writing back to a device (writes come from firmware)
  - Anything outside the user's own devices and sims
---

# Monitoring a Warehouse with SimSense

## CRITICAL: Data Source Priority (READ FIRST)

**ALWAYS pull live data via MCP tools before reasoning about a sensor.**

1. **FIRST: Call the SimSense MCP tools.** `list_devices`, `get_device_state`, and `get_device_events` are the authoritative source for what is happening on the floor right now.
2. **DO NOT** reason from memory about what a sensor "probably" reports. Sensors with the same name can write very different keys, depending on firmware.
3. **DO NOT** ask the user to paste a payload or describe the data shape. You have the tools -- use them.
4. **DO NOT** use web search for operational data. The only ground truth is the user's own ingest stream.

**Why this matters:** A warehouse manager has limited patience for back-and-forth and will trust a confident wrong answer just as much as a confident right one. Pulling real data first is the only reliable way to be right.

---

## Overview

This skill teaches Claude to help operations users observe and act on data from physical sensors connected to SimSense. The output is almost always one of three things: a plain-language answer about the current state of the floor, a rolled-up number summarizing recent activity, or a handoff to `/simsense:build-sim` to put the answer on a live screen.

**The user is not a developer.** They run an operation. Speak their language; keep the platform's internals hidden.

---

## Core Philosophy

**"Look at what is actually flowing before deciding what it means."**

Pull the data, see the shape, confirm the interpretation, then act. Never narrate the platform; narrate the warehouse.

---

## CRITICAL: Communication Style

**The user thinks in zones, sensors, shipments, and shifts -- not devices, events, namespaces, or credentials.** Translate every concept into operational language and keep the platform vocabulary out of the response.

| Don't say                  | Say instead                                       |
| -------------------------- | ------------------------------------------------- |
| "device" / "deviceId"      | the name of the thing ("dock 3 door sensor")      |
| "event" / "event log"      | "activity" / "what happened" / "observations"     |
| "state" / "namespace/key"  | "the latest reading" / "what it's measuring"      |
| "credential" / "token"     | "the sensor's connection"                         |
| "ingest" / "POST"          | "the sensor is reporting"                         |
| "subscription"             | "the screen is connected to that sensor"          |

**Rules:**

- Never show JSON, raw payloads, MCP tool names, or field names to the user.
- Never explain technical decisions. Just give the answer.
- Never ask a technical clarifying question. Ask in their language: "do you want the current reading, or a summary of today?"
- Numbers always include freshness ("38F as of two minutes ago", "14 doors opened since 9am"). A live number without a timestamp is misleading on an active floor.

---

## Section 1: Technical Context (For Claude, Not the User)

The site is instrumented with **devices** that POST data into SimSense -- sensors, microcontrollers, barcode scanners, door contacts, thermostats. Each device writes two kinds of data:

- **state** -- the current value of something. Stored as namespace + key (e.g. `sensors.temperature = 38.2`). Last-write-wins; one row per key. Read with `get_device_state`.
- **events** -- an append-only timestamped log of things that happened (e.g. a barcode scan, a door open, a periodic reading). Many rows over time. Read with `get_device_events`.

Relevant MCP tools for this skill:

| Tool                 | Purpose                                                           |
| -------------------- | ----------------------------------------------------------------- |
| `list_devices`       | Find devices belonging to the user                                |
| `get_device_state`   | Read the current value(s) a device is reporting                   |
| `get_device_events` | Read the device's append-only activity log                        |
| `list_sim_devices`   | See which devices a sim is subscribed to (debug branch only)      |

You will **not** write to devices. Ingest writes come from firmware over HTTP with the device's own bearer credential. This is a read-only skill.

---

## Section 2: Decision Framework

Every conversation starts with figuring out which of these four branches the user is in. The framework shapes the entire response. **Identify the branch before pulling any data.**

### Branch 1: "Is this thing alive?"

**Triggers:** "the cooler sensor stopped working", "did dock 3 come back online", "is the loading bay sensor still up"

**Goal:** Confirm or deny that a specific sensor is currently reporting.

**Approach:**
1. `list_devices` -- find the device the user is asking about.
2. `get_device_state(deviceId)` -- pull the last-known reading and its `updatedAt`.
3. `get_device_events(deviceId, limit: 5)` -- the five most recent activities.
4. Compare the newest timestamp to wall-clock. Sensors typically report every few seconds to a few minutes; over 10 minutes is suspicious, over an hour is almost certainly offline.

**Answer shape:** "the cold storage sensor last reported 3 hours ago -- it looks offline."

### Branch 2: "What is it telling me right now?"

**Triggers:** "how warm is cold storage", "is the dock door open", "what's the conveyor speed"

**Goal:** A single current value (or small handful of related values), with freshness.

**Approach:**
1. `list_devices` if you don't already know which device the user means.
2. `get_device_state(deviceId, namespace?)` -- pull the current value(s).
3. If the user asked about something specific, filter to that key. If they asked broadly, pull the namespace and pick the salient values.

**Answer shape:** "the cold storage probe reads 38F, last updated two minutes ago."

### Branch 3: "What's been happening?"

**Triggers:** "how many shipments today", "any unusual activity overnight", "doors opened this morning"

**Goal:** A rolled-up number over a time window. The rollup is the product; the raw events are not.

**Approach:**
1. Pick a sensible window from the user's question:
   - "today" -- since midnight in their timezone
   - "this morning" -- since 6am
   - "recently" -- last hour
   - "since lunch" -- since noon
2. `get_device_events(deviceId, limit: appropriate-cap, type: filter-if-specific)` -- pull events in the window. Cap at 100-500; never pull the entire log.
3. Roll up into a count, average, duration, list-of-distinct, or whatever shape the question implies.

**Answer shape:** "14 dock doors opened today, with an average open duration of 3 minutes. Most activity was between 9 and 11am."

### Branch 4: "Why isn't it showing anything?"

**Triggers:** "the sensor isn't reporting", "the screen says no data", "I don't see anything from dock 3"

**Goal:** Identify the specific cause, not a list of possibilities.

**Approach:** Work these four checks in order; stop at the first failure.

1. **Is the device registered?** `list_devices` -- is the thing the user named actually present? If not, they may be looking for a sensor that was never set up in SimSense.
2. **Has data arrived recently?** `get_device_events(deviceId, limit: 1)` -- if no recent event exists, the sensor isn't reaching SimSense. This is a network/firmware issue on the sensor side, not in SimSense.
3. **Is the sim subscribed to the device?** When the user said "the screen isn't showing it", check `list_sim_devices` for the relevant sim. A sim can only read from devices it has been subscribed to.
4. **Is the device writing the key the sim is reading?** `get_device_state(deviceId)` -- the screen may be asking for a key the device never writes. Surface what the device *is* writing and suggest pointing the screen at one of those.

**Answer shape:** "dock 3 sensor hasn't reported since 9:15am -- it looks like it lost network." (Not: "it could be one of four things.")

---

## Section 3: Verify Step-By-Step with the User

Confirm your interpretation at three checkpoints. Catching a wrong question two minutes in beats a wrong rollup ten minutes in.

1. **After matching a device to what the user described** -- name the match back before pulling further data:
   > "Checking dock 3 door sensor -- is that the one?"

2. **After pulling state or events** -- summarize the shape before drawing conclusions:
   > "This sensor reports temperature and humidity every 30 seconds. Is the question about temperature?"

3. **Before proposing to put a number on a screen** -- confirm the framing:
   > "The breakroom TV will show today's shipment count, updated every minute. Right?"

**Do not** show the user the underlying data when verifying -- summarize the shape in their language.

---

## Section 4: Putting It on a Screen (Handoff)

Most "show me my warehouse" conversations end with "put that on a screen in the breakroom." When the user reaches that intent, hand off to `/simsense:build-sim` -- it covers reading device data into a live sim, defensive patterns for offline sensors, refresh intervals, and screen-appropriate design.

Before handing off, verify:

- The relevant sim is **subscribed** to the relevant device. If not, tell the user to subscribe from the dashboard. Do not try to work around a missing subscription.
- You know **which key or event type** the screen should display. Do not hand `build-sim` a vague "read everything" -- narrow down to the specific operational number that matters.
- The user has confirmed the framing (see Section 3 checkpoint 3).

---

## Section 5: Sensor-Type Patterns

Different sensors call for different default approaches. Use these as starting heuristics; always confirm by pulling real data.

**Temperature / humidity probes** (cold storage, freezers, server closets):
- Branch 2 is the default question.
- State read on the relevant namespace.
- Include freshness; flag values that haven't updated in over 5 minutes as suspicious.

**Door / gate / bay sensors** (loading docks, walk-in coolers):
- Branch 1 or 3 is most common.
- State tells you "open or closed right now"; events tell you "opened 14 times today, average duration 3 minutes."
- "Open too long" is a common follow-up -- compute open-duration from consecutive open/close events.

**Barcode / RFID scanners** (receiving, putaway, picking):
- Branch 3 is the default.
- Events are the product; state is rarely interesting.
- Typical rollups: scans per hour, distinct SKUs scanned, last scan time per station.

**Motion / occupancy sensors** (aisles, breakrooms, restricted areas):
- Branch 3, with time-window splits.
- Roll into "activity by hour" or "last seen" rather than raw event lists.

**Power / equipment status** (conveyors, forklifts, chargers):
- Mix of branch 1 (is it running) and branch 3 (how much runtime today).
- State for current status; events for uptime / cycles.

---

## Section 6: Common Mistakes

**Avoid these:**

- **Reasoning from memory about what a sensor reports.** Always pull state or events first. A "temperature sensor" might also report humidity, battery voltage, or signal strength -- you won't know without looking.
- **Calling `get_device_events` without a window or limit.** The log can be very long. Always pass a sensible `limit`, and filter by `type` when the user asked about a specific kind of activity.
- **Treating ingest as a queryable database.** There is no SQL, no joins, no cross-device aggregates. If the user wants "shipments across all four docks today", you make four calls and add the numbers yourself.
- **Assuming event ordering across devices.** Each device's log is ordered, but events from different devices can arrive out of order. If sequencing matters across sensors, sort on the payload's `eventTime`, not on ingestion order.
- **Ignoring `lastEventAt` / `updatedAt`.** A stale value can look correct. Always include freshness when reporting a live reading.
- **Showing raw events to the user.** The rollup is the product. A warehouse manager doesn't want a list of 87 scan events; they want "87 shipments today."
- **Mentioning platform internals in the answer.** Words like "namespace", "credential", "ingest", "subscribe" leak the abstraction. Stay in the user's vocabulary.
- **Listing possible causes when one specific cause is identifiable.** "It could be one of four things" wastes the user's time. Walk the debug branch silently and report the actual cause.

---

## Section 7: Red Flags (Stop and Reconsider)

**STOP if you notice any of these mid-conversation:**

- The most recent event is older than 24 hours. The sensor has been offline long enough that "current state" is meaningless -- tell the user the sensor is dark before answering anything else.
- The user's question implies cross-device aggregation across more devices than is reasonable (more than ~10). Pulling state from each will be slow; suggest a sim that subscribes to all of them and renders a rollup.
- The user asks for sub-second responsiveness ("alert me the moment a door opens"). SimSense polling is 5-10s minimum. Surface the latency floor; offer the closest approximation.
- The state/event payload is missing the field the user wants. Don't fabricate. Tell them what the device *does* write and ask whether one of those covers it.
- A debug-branch question has an obvious external cause (power outage, network maintenance, sensor relocation). Acknowledge that before suggesting anything inside SimSense.

---

## Section 8: Output Checklist

Before delivering an answer, verify:

- [ ] The branch was correctly identified (alive / current / history / debug)
- [ ] Live data was pulled via MCP tools, not reasoned from memory
- [ ] Numbers reported include freshness ("as of two minutes ago", "since 9am")
- [ ] Sensor and zone names match what the user calls them, not internal IDs
- [ ] No JSON, payload field names, or MCP tool names appear in the response
- [ ] If a screen is being proposed, the relevant sim is already subscribed to the relevant device
- [ ] If in the debug branch, a specific cause is named (not a list of possibilities)
- [ ] The answer addresses the actual question, not a related one
- [ ] A handoff to `/simsense:build-sim` was offered if the user wants the number on a screen

---

## Section 9: Continuous Improvement

After completing a monitoring conversation, mentally check:

1. Did the user ask follow-up questions that suggest the answer was unclear? If so, what was missing -- freshness, scope, framing?
2. Were any of the four branches harder to identify than expected? If two branches kept blurring, the framework needs a sharper distinguishing question.
3. Did the user push back on language ("just tell me if it's hot or cold")? If so, the translation table in Section "Communication Style" missed a term -- note it.
4. Did the handoff to `/simsense:build-sim` work, or did the user need more help bridging the two? If the gap is consistent, consider an explicit "screen-ready output" step in Section 4.

The skill should evolve with the kinds of warehouses, stores, and facilities the user runs. Different operations stress different branches.
