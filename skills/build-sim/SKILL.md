---
name: build-sim
description: >
  Best practices for building high-quality SimSense sims. Use this skill whenever the user wants to create or update a sim, asks Claude to build something for a
  screen or display, mentions SimSense HTML, or wants to use the SimSense SDK for state, events, or real-time features. Also use when improving an existing
  sim's design, layout, or interactivity.
---

# Building SimSense Sims

## Communication style

SimSense users are typically non-technical. They think in terms of "what I want on my screen", not HTML or JavaScript. Never show code in your responses,
explain technical decisions, or ask technical questions. Talk about colors, layouts, content, and behavior -- not divs, CSS, or APIs. Just build it and show
them the live URL.

## Technical context (for Claude, not the user)

A sim is a self-contained HTML page deployed to a live URL and displayed on physical screens. Build each sim as a single HTML file with inline CSS and JS.

## Design for screens, not browsers

Sims run on TVs, tablets, kiosks, and lobby displays. Before building, ask the user about their target screen: **Is it touch-enabled or passive? What
size/orientation? Where is it located?** This changes the design significantly - a touch kiosk on a cafe counter needs big tap targets and interactive flows,
while a wall-mounted TV needs auto-rotating content that works unattended.

Design accordingly:

- **Landscape by default.** Most screens are 16:9. Use `100vw` / `100vh` and avoid scroll.
- **Large, readable text.** Someone viewing from 3-10 feet away. Minimum effective font size is much larger than web - think 24px+ for body text, 48px+ for
  headings.
- **High contrast.** Screens in bright lobbies, cafe counters, conference rooms. Ensure readability in ambient light.
- **No tiny interactive targets.** If touch-enabled (kiosk, tablet), buttons should be at least 48x48px with generous spacing.
- **Graceful without interaction.** Many sims run on passive displays. Design so the sim is useful and visually interesting even if nobody touches it.
- **Avoid scroll.** Content should fit the viewport. If you have more content than fits, rotate or paginate it on a timer.

## HTML structure

Always produce a complete, valid HTML document:

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Sim Title</title>
    <style>
      /* All CSS inline */
      * {
        margin: 0;
        padding: 0;
        box-sizing: border-box;
      }
      body {
        width: 100vw;
        height: 100vh;
        overflow: hidden;
      }
    </style>
  </head>
  <body>
    <!-- Content -->
    <script>
      // All JS inline
    </script>
  </body>
</html>
```

External **images, fonts, and CSS libraries** (Google Fonts, Unsplash, animate.css, etc.) are fine and encouraged. Keep JS inline -- do not import JS frameworks
from CDNs.

## SimSense SDK

The SDK is automatically injected into every sim as `window.SimSense`. You don't need to include it - it's available at runtime. All methods are async.

### Persistent state

State is shared across all viewers of a sim and persists across page reloads. Use it for anything the sim needs to remember: scores, votes, settings,
user-submitted data.

```js
await SimSense.set("data", "score", 42);
const score = await SimSense.get("data", "score"); // 42
const all = await SimSense.getAll("data"); // { score: 42 }
await SimSense.delete("data", "score");
```

The first argument is always a **namespace** - use it to group related keys (e.g., `'votes'`, `'settings'`, `'players'`).

**Atomic counters** - for votes, view counts, or any concurrent increment:

```js
const result = await SimSense.increment("votes", "option-a");
const count = result.value; // Always extract .value - don't use the result directly
```

### Events

An append-only log for recording things that happen: button presses, form submissions, sensor readings. Events are timestamped and queryable.

```js
await SimSense.emit("button_press", { buttonId: "start", user: "anon" });

const events = await SimSense.getEvents("button_press", { limit: 50 });
// [{ id, type, data: { buttonId, user }, createdAt }, ...]

const { count } = await SimSense.countEvents("button_press");
```

### Configuration

Read-only from the sim's perspective. Config is set externally via the `set_sim_config` MCP tool. Use this for values the sim owner wants to change without
editing HTML - menu items, poll options, display preferences, API endpoints.

```js
const theme = await SimSense.getConfig("theme"); // single key
const allConfig = await SimSense.getConfig(); // all config
```

When building interactive sims, prefer config over hardcoded values. For example, a voting sim should read its options from config so the user can change them
via `set_sim_config` without touching the HTML.

### Real-time sync

For live updates across multiple screens viewing the same sim:

```js
// Fires whenever any key in 'scores' namespace changes
const unsub = SimSense.subscribe("scores", (key, value) => {
  document.getElementById(key).textContent = value;
});

// Fires on new events of a given type
const unsub2 = SimSense.onEvent("new_order", (data) => {
  addOrderToQueue(data);
});
```

These use WebSocket connections under the hood. Always handle the case where the connection drops - the SDK reconnects automatically, but your UI should be
resilient to brief gaps.

## Tags

When creating or updating a sim, pick appropriate tags from this list:

`dashboard`, `game`, `voting`, `art`, `menu`, `weather`, `music`, `education`, `ambient`, `info`, `interactive`, `countdown`, `photo`, `sports`, `status`,
`schedule`

Tags help users discover sims on the showcase. Pick 1-3 that best describe the sim.

## Common patterns

**Auto-rotating content:** For menu boards, news tickers, or galleries:

```js
let index = 0
const items = [...] // your content array
setInterval(() => {
  index = (index + 1) % items.length
  render(items[index])
}, 8000)
```

**Day/night themes:** Adjust colors based on time of day:

```js
const hour = new Date().getHours();
const isDark = hour < 6 || hour > 20;
document.body.classList.toggle("dark", isDark);
```

**Periodic data refresh:** For dashboards pulling from APIs:

```js
async function refresh() {
  const data = await fetch("https://api.example.com/data").then((r) => r.json());
  render(data);
}
refresh();
setInterval(refresh, 60000);
```

**Config-driven content:** Let the user control what's displayed without editing HTML:

```js
async function init() {
  const items = await SimSense.getConfig("items"); // set via set_sim_config
  const title = await SimSense.getConfig("title");
  render(title, items);
}
init();
```

## What NOT to do

- Don't use React, Vue, or any framework - raw HTML/CSS/JS only
- Don't import JS frameworks from CDNs (images, fonts, and CSS libraries are fine)
- Don't build your own backend or database - use SimSense state and events
- Don't assume mouse hover - many displays are passive or touch-only
- Don't use small text or low-contrast colors - screens are viewed from a distance
- Don't create scrollable layouts - paginate or rotate content instead
