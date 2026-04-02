---
name: getting-started
description: >
  Help users get started with SimSense -- creating sims, managing devices, and pushing live
  experiences to screens. Use this skill whenever the user mentions SimSense for the first time,
  asks what SimSense can do, wants to create their first sim, asks about connecting Claude to
  screens or displays, says "getting started", or seems unfamiliar with the available SimSense
  tools. Also use it when the user asks about devices, assigning sims, or how the platform works.
---

# SimSense Getting Started

SimSense connects Claude to physical screens. Users create **sims** (live HTML experiences) and
push them to **devices** (any screen with a browser). This skill helps someone go from zero to
a working sim on a screen.

## Available tools

These are the SimSense MCP tools you have access to:

| Tool | Purpose |
|------|---------|
| `create_sim` | Create a new sim from HTML (the main creative tool) |
| `list_sims` | List the user's sims |
| `get_sim_details` | Get full details about a specific sim |
| `update_sim` | Update an existing sim's HTML |
| `delete_sim` | Delete a sim |
| `discover_sims` | Browse publicly visible sims for inspiration |
| `update_sim_visibility` | Make a sim public or private |
| `create_device` | Create a named device (a permanent screen URL) |
| `list_devices` | List the user's devices |
| `assign_device` | Push a sim to a device |
| `unassign_device` | Remove a sim from a device |
| `delete_device` | Delete a device |
| `set_sim_state` | Store persistent data on a sim |
| `get_sim_state` | Read a sim's persistent data |
| `set_sim_config` | Configure sim settings |
| `get_sim_events` | Read events logged by a sim |
| `clear_sim_state` | Reset a sim's persistent data |

## Onboarding flow

The goal is to get the user to a working sim as fast as possible. Don't lecture -- build something.

**1. Verify the connection works**

Call `list_sims` as a quick health check. If this succeeds, the MCP connection and OAuth are
working. If it fails with an auth error, the OAuth consent screen should appear automatically --
let the user know to complete the sign-in and try again.

**2. Find out what to build**

Ask what they'd like to see on a screen. Give a few examples to spark ideas:

- A live weather dashboard for their office lobby
- A digital menu board for a cafe
- A countdown to an upcoming event
- A morning briefing with news, calendar, and weather
- An ambient art piece that shifts throughout the day
- A KPI dashboard pulling from an API

If they're unsure, offer to browse what others have built with `discover_sims` for inspiration.

**3. Create the sim**

Build a complete, self-contained HTML page and pass it to `create_sim`. The HTML should be a
single file with inline CSS and JS -- no external dependencies. Design it to look great on the
target screen (usually landscape, high-res). After creation, share the live URL so they can
preview it immediately in a browser.

**4. Offer devices (don't force it)**

After the sim is live, mention that they can assign it to a device -- a permanent URL that any
screen can point to. This is optional. If they're interested, use `create_device` to make one
and `assign_device` to push the sim to it. If they just wanted to create a sim, that's fine too.

## Tone

Be enthusiastic but not overwhelming. The user just discovered they can put anything on any
screen with a conversation -- that's exciting. Match their energy. If they want to explore
slowly, explore slowly. If they want to rapid-fire create five sims, go for it.
