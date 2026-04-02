# SimSense Plugin for Claude Code

Connect Claude to [SimSense](https://simsense.ai) -- create sims, manage
devices, and push live experiences to any screen.

## Install

```
claude plugin install simsense@simsenseai
```

Or add the marketplace to Claude Code:

```
claude plugin marketplace add simsenseai/simsense-plugin
```

Then install from the `/plugin` menu.

## What it does

This plugin auto-configures the SimSense MCP server so Claude can:

- **Create sims** -- interactive HTML experiences deployed to `*.my.simsense.ai`
- **Manage devices** -- assign sims to permanent screen URLs
- **Read and update sim state** -- persistent data, config, and events
- **Discover sims** -- browse the public sim showcase

No manual MCP configuration needed. OAuth authentication is handled
automatically on first use.

## Skills

| Skill                       | Description                                                  |
| --------------------------- | ------------------------------------------------------------ |
| `/simsense:getting-started` | Guided walkthrough for creating your first sim               |
| `/simsense:build-sim`       | Best practices for screen design, SimSense SDK, and patterns |

## Links

- [simsense.ai](https://simsense.ai) -- Marketing site
- [my.simsense.ai](https://my.simsense.ai) -- Platform dashboard
- [FAQ](https://simsense.ai/faq) -- Common questions
