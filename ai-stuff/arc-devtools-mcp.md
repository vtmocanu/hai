# Arc Browser MCP

I wanted Claude Code to interact with my browser: read page content, open tabs, click elements, take screenshots. The [arc-devtools-mcp](https://github.com/fukayatti/arc-devtools-mcp) server makes this possible for Arc Browser.

## What It Does

The Arc DevTools MCP server connects Claude Code to Arc Browser via the Chrome DevTools Protocol (CDP). Once connected, Claude can:

- **List open tabs** and switch between them
- **Open new pages** and navigate to URLs
- **Read page content** via accessibility tree snapshots
- **Click elements**, fill forms, press keys
- **Take screenshots** of pages or elements
- **Monitor network requests** and console messages

It's a fork of Google's [Chrome DevTools MCP](https://github.com/ChromeDevTools/chrome-devtools-mcp), modified to avoid `Target.createTarget` which crashes Arc due to conflicts with its space/tab management.

## What Does NOT Work

### The `npx` package is broken

The README suggests:

```json
{
  "mcpServers": {
    "arc-devtools": {
      "command": "npx",
      "args": ["-y", "@fukayatti0/arc-devtools-mcp"]
    }
  }
}
```

This fails because the `chrome-devtools-frontend` npm dependency ships TypeScript source files (`.ts`) but the compiled code imports `.js` files. The module resolution breaks with:

```
Error [ERR_MODULE_NOT_FOUND]: Cannot find module 'chrome-devtools-frontend/mcp/mcp.js'
```

### Arc's built-in remote debugging checkbox

Arc has a setting at `arc://inspect/#remote-debugging` with a checkbox: "Allow remote debugging for this browser instance". Enabling this shows "Server running at: 127.0.0.1:9222", but **the CDP HTTP endpoint doesn't actually work reliably**. Connection gets refused after browser restarts, and the MCP server can't connect.

## What Works: Build from Source + CLI Flag

### 1. Clone and build locally

```bash
git clone https://github.com/fukayatti/arc-devtools-mcp.git
cd arc-devtools-mcp
pnpm install
npm run build
```

The build emits TypeScript errors (from the `chrome-devtools-frontend` dependency) but uses `tsc || true`, so it continues and produces working JavaScript output in `build/`.

### 2. Launch Arc with remote debugging

Quit Arc completely, then start it from the terminal:

```bash
/Applications/Arc.app/Contents/MacOS/Arc --remote-debugging-port=9222 &
```

This opens the CDP endpoint at `http://127.0.0.1:9222`. You can verify it works:

```bash
curl -s http://127.0.0.1:9222/json/version | jq .Browser
# "Chrome/146.0.7680.76"
```

### 3. Configure Claude Code MCP

Point the MCP server to the local build with the `--browserUrl` flag:

```bash
claude mcp add arc-devtools -s user -- \
  node /path/to/arc-devtools-mcp/build/src/main.js \
  --browserUrl http://127.0.0.1:9222
```

Restart Claude Code, and the arc-devtools tools should show as connected:

```
arc-devtools: node /path/to/arc-devtools-mcp/build/src/main.js --browserUrl http://127.0.0.1:9222 - Connected
```

## Available Tools

Once connected, these MCP tools become available:

| Tool | Purpose |
|------|---------|
| `list_pages` | List all open tabs |
| `select_page` | Switch to a tab (with `bringToFront`) |
| `new_page` | Open a new tab with a URL |
| `navigate_page` | Navigate current tab (URL, back, forward, reload) |
| `take_snapshot` | Get page content as accessibility tree (text) |
| `take_screenshot` | Capture page as PNG/JPEG/WebP |
| `click` | Click an element by uid |
| `fill` | Fill a form field |
| `press_key` | Send keyboard input |
| `hover` | Hover over an element |
| `evaluate_script` | Run JavaScript on the page |

## Usage Example

A typical interaction flow:

1. Ask Claude to **list pages** to see what's open
2. **Select a page** and bring it to front
3. **Take a snapshot** to get the accessibility tree with element uids
4. **Click** on elements using their uid, or **fill** form fields
5. **Navigate** to new URLs or **open new pages**

```
You: check my current arc tab
Claude: [calls list_pages] → Your current tab is localhost:1313/my-blog-post

You: click the "Submit" button
Claude: [calls take_snapshot] → finds button uid=1_42
Claude: [calls click uid=1_42] → Successfully clicked
```

## Limitations

- **No Arc Spaces control**: CDP doesn't know about Arc's space concept. New tabs open in whichever space is currently active
- **Requires terminal launch**: Arc must be started with `--remote-debugging-port=9222` for reliable CDP access
- **Security**: Remote debugging gives full control over the browser, including access to cookies, site data, and navigation. Only use on trusted machines

{{< callout type="warning" >}}
Remote debugging exposes full browser control on port 9222. Don't enable this on shared or untrusted networks.
{{< /callout >}}

## Related

- [What is MCP?]({{< ref "mcp-intro" >}}) - Overview of the Model Context Protocol
- [Commands vs MCP vs Skills]({{< ref "commands-vs-mcp-vs-skills" >}}) - How MCP fits into the Claude Code extension model

