# chrome-cdp

Let your AI agent see and interact with your **live browser sessions** — Chrome, Dia, Brave, Edge, and Arc. Connects to the tabs you already have open, across multiple browsers and profiles simultaneously.

No Puppeteer, no separate browser instance, no re-login. One WebSocket connection per browser, multiplexed across all tabs.

> Fork of [pasky/chrome-cdp-skill](https://github.com/pasky/chrome-cdp-skill) with multi-browser, multi-profile, and master daemon architecture.

## What's different from upstream

| Feature | upstream (pasky) | this fork |
|---------|-----------------|-----------|
| Browsers | Chrome only | Chrome, Dia, Brave, Edge, Arc, Chromium |
| Architecture | One daemon per tab | **Master daemon** per browser port |
| "Allow" popups (Chrome) | Every tab | **Once** per session |
| Multi-agent | No | Yes — shared master daemon via Unix socket |
| Profiles | No | List, filter, open in specific profile |
| Session persistence | No | `cdp use 9223` sets active browser |
| Browser switching | No | `--port` / `--browser` flags |

## Installation

```bash
npx skills add seahyc/chrome-cdp-skill@chrome-cdp -g -y
```

### Enable remote debugging

| Browser | How to enable | Default port |
|---------|--------------|--------------|
| **Chrome** | Toggle at `chrome://inspect/#remote-debugging` | 9222 |
| **Dia** | Launch with `--remote-debugging-port=9223` | 9223 |
| **Brave** | Launch with `--remote-debugging-port=9224` | 9224 |
| **Edge** | Launch with `--remote-debugging-port=9225` | 9225 |
| **Arc** | Launch with `--remote-debugging-port=9227` | 9227 |

Requires **Node.js 22+** (uses built-in WebSocket). No npm install needed.

## Quick start

```bash
# List tabs in Chrome
scripts/cdp.mjs --port 9222 list

# List tabs in Dia
scripts/cdp.mjs --port 9223 list

# Evaluate JS in a tab
scripts/cdp.mjs --port 9222 eval <target> "document.title"

# Screenshot
scripts/cdp.mjs --port 9222 shot <target>
```

## Multi-browser support

```bash
# Target by port (recommended)
scripts/cdp.mjs --port 9222 list    # Chrome
scripts/cdp.mjs --port 9223 list    # Dia

# Target by name (reads DevToolsActivePort file)
scripts/cdp.mjs --browser chrome list
scripts/cdp.mjs --browser dia list

# Session persistence (single-agent use)
scripts/cdp.mjs use 9223            # Set Dia as default
scripts/cdp.mjs list                # Now targets Dia
scripts/cdp.mjs use auto            # Clear

# Environment variables
CDP_PORT=9223 scripts/cdp.mjs list
CDP_BROWSER=dia scripts/cdp.mjs list
```

## Profile support

All Chrome profiles share the same debug port. Tabs from all profiles appear in `list`.

```bash
# List all profiles
scripts/cdp.mjs --port 9222 profiles
#   ● Casual Me (Ying Cong Seah)  [Profile 2]
#   ● Old Friend (Seah Ying Cong)  [Profile 4]
#   ○ Work  [Default]

# List shows profile column
scripts/cdp.mjs --port 9222 list
# D8D3BE54  [Casual Me   ]  Example Page  https://example.com

# Filter by profile
scripts/cdp.mjs --port 9222 list --profile "Casual Me"

# Open URL in default profile
scripts/cdp.mjs --port 9222 open "https://example.com"

# Open in a specific profile (works even if that profile window isn't open yet)
scripts/cdp.mjs --port 9222 open "https://example.com" --profile "Old Friend"
```

## Commands

```bash
scripts/cdp.mjs list [--profile <name>]           # list open pages
scripts/cdp.mjs profiles                           # list browser profiles
scripts/cdp.mjs open <url> [--profile <name>]      # open new tab

scripts/cdp.mjs snap   <target>                    # accessibility tree
scripts/cdp.mjs eval   <target> <expr>             # evaluate JS
scripts/cdp.mjs shot   <target> [file]             # screenshot
scripts/cdp.mjs html   <target> [selector]         # get HTML
scripts/cdp.mjs nav    <target> <url>              # navigate
scripts/cdp.mjs net    <target>                    # network timing
scripts/cdp.mjs click  <target> <selector>         # click by CSS selector
scripts/cdp.mjs clickxy <target> <x> <y>           # click at coordinates
scripts/cdp.mjs type   <target> <text>             # type text
scripts/cdp.mjs loadall <target> <selector> [ms]   # click until gone
scripts/cdp.mjs evalraw <target> <method> [json]   # raw CDP command
scripts/cdp.mjs stop   [target]                    # stop daemon
scripts/cdp.mjs use    <browser|port|auto>         # set active browser
```

`<target>` is a unique prefix of the targetId shown by `list`.

## Architecture: master daemon

A single **master daemon** per browser port holds one WebSocket connection and multiplexes CDP sessions across all tabs:

```
CLI invocation
  └─► connects to /tmp/cdp-master-<port>.sock
        └─► Master Daemon (one per port)
              ├─ holds single browser WebSocket
              ├─ sessions Map<targetId, sessionId>
              └─ lazy attach/detach per tab
```

- Chrome's "Allow debugging?" popup fires **once** when the daemon starts
- Multiple agents connect simultaneously — concurrent requests across different tabs
- Daemon auto-exits after 20 min idle
- Protocol: newline-delimited JSON over Unix socket

### Multi-agent usage

When multiple agents run concurrently, use `--port` flags directly:

```bash
# Agent A
scripts/cdp.mjs --port 9222 eval D8D3BE54 "document.title"

# Agent B (same time, different tab)
scripts/cdp.mjs --port 9222 snap 23B3B848
```

Don't use `cdp use` with concurrent agents — the session file is shared.

## Chrome notes

- **Chrome 146+** ignores `--remote-debugging-port` on the default profile. Use the `chrome://inspect` toggle instead.
- **Dia, Brave, Edge** work with `--remote-debugging-port` directly — no popup at all.
- Only **loaded** tabs appear in `list`. Suspended tabs are invisible until clicked in the browser.

## Credits

Based on [pasky/chrome-cdp-skill](https://github.com/pasky/chrome-cdp-skill) by Petr Baudis.
