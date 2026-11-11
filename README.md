# pi-web-fetch

A [pi](https://github.com/badlogic/pi-mono) extension that gives your agent a proper web browser. Fetches pages via a headless Chromium you already have, extracts clean content via [trafilatura](https://trafilatura.readthedocs.io/), and optionally distills it with an LLM.

This is the KessokuNet fork of [georgebashi/pi-web-fetch](https://github.com/georgebashi/pi-web-fetch). It drops puppeteer's bundled Chromium download in favour of `puppeteer-core` driving a system browser (Helium, Chromium, Chrome), which is configurable and roughly 400 MB lighter.

## Why not use Claude Code's built-in WebFetch?

Claude Code's WebFetch uses Turndown to convert HTML to markdown — a simple regex-based converter that can't handle JavaScript-rendered pages, doesn't strip boilerplate well, and has no extensibility. pi-web-fetch improves on this in several ways:

| | Claude Code WebFetch | pi-web-fetch |
|---|---|---|
| **Rendering** | Static HTTP fetch | Headless Chrome (handles SPAs, JS-rendered content) |
| **Extraction** | Turndown (regex HTML→md) | [trafilatura](https://trafilatura.readthedocs.io/) (ML-based boilerplate removal) |
| **Raw content** | Never — prompt is mandatory | Optional — omit prompt to get full markdown |
| **Batch fetching** | One URL at a time | Up to 10 URLs concurrently with per-URL progress |
| **Concurrency** | Sequential | Browser pool with 6 parallel tabs |
| **Extensibility** | None | Hook system for site-specific handling |
| **Smart redirects** | Generic | Context-aware (e.g. GitHub URLs → `gh` CLI suggestions) |

## Features

- **`web_fetch` tool** — registered in pi's tool system, callable by the LLM
- **Headless Chromium** via `puppeteer-core` — handles JavaScript-rendered pages, SPAs, pages behind cookie banners, using a browser already installed on the machine
- **Content extraction** — strips boilerplate (nav, ads, footers) using trafilatura's ML-based extraction, outputs clean markdown
- **LLM processing** — optionally distills page content to answer a specific question via a pi sub-agent
- **Batch fetching** — fetch up to 10 pages concurrently in a single tool call with per-URL status in the UI
- **Browser pool** — reuses a single browser instance with up to 6 parallel tabs, avoiding repeated browser startup overhead
- **Smart large-page handling** — when content exceeds ~50KB and no prompt is given, automatically generates a structured summary
- **15-minute cache** — avoids redundant fetches; enables summarize-then-drill-down workflows
- **Cross-host redirect detection** — reported to the LLM for explicit follow-up rather than silently following
- **HTTP→HTTPS auto-upgrade**
- **Extension hooks** — customize fetch behavior for specific sites (redirect, replace HTML, transform markdown, override summarization)
- **Built-in site handlers** — GitHub URLs redirect to `gh` CLI, Google Docs redirect to workspace tools

## Prerequisites

- [pi](https://github.com/badlogic/pi-mono) coding agent
- A Python tool runner for trafilatura (auto-detected in priority order):
  1. [uv](https://docs.astral.sh/uv/) (`uvx`) — fastest, recommended
  2. `uv run` — fallback if `uvx` alias is missing
  3. [pipx](https://pipx.pypa.io/) — widely available on Debian/Ubuntu
  4. [pip-run](https://github.com/jaraco/pip-run) — niche fallback
- Node.js 18+

- Node.js 18+
- A Chromium-based browser: `/usr/lib64/helium-browser-bin/helium`, `/usr/bin/helium`, `/usr/bin/chromium`, `/usr/bin/chromium-browser`, `/usr/bin/google-chrome-stable`, `/usr/bin/google-chrome`, or `/snap/bin/chromium` (first match wins). Override with `PUPPETEER_EXECUTABLE_PATH` or the `browserExecutable` config key.

## Installation

```bash
pi install git:github.com/KessokuNet/pi-web-fetch
```

Or clone for development:

```bash
git clone https://github.com/KessokuNet/pi-web-fetch
cd pi-web-fetch
npm install
```

> **Note:** `npm install` does not download Chromium. Set `PUPPETEER_EXECUTABLE_PATH` if none of the paths above exist on your system.

> **Note:** The first time `web_fetch` runs, `uvx` will download the trafilatura package (~10MB). Subsequent runs use the cached environment and are fast.

### Add to pi

Add the package to your pi settings (`~/.pi/agent/settings.json`):

```json
{
  "packages": [
    "git:github.com/KessokuNet/pi-web-fetch"
  ]
}
```

Or point to a local checkout:

```json
{
  "packages": [
    "../../Projects/pi-web-fetch"
  ]
}
```

Or use the `-e` flag for quick testing:

```bash
pi -e git:github.com/KessokuNet/pi-web-fetch
```

## Usage

The extension registers a `web_fetch` tool. The LLM will use it automatically when it needs to fetch web content.

### Single URL

**With a prompt (recommended):**
> Fetch https://docs.example.com/api and tell me what authentication methods are supported.

**Without a prompt (full content):**
> Fetch the content of https://example.com/changelog

### Batch fetching

Fetch multiple pages concurrently by asking for several URLs at once:
> Read these three pages and compare their approaches to error handling:
> - https://docs.python.org/3/tutorial/errors.html
> - https://go.dev/blog/error-handling-and-go
> - https://doc.rust-lang.org/book/ch09-00-error-handling.html

The agent will use the `pages` parameter to fetch all URLs in parallel (up to 10 per call). The UI shows live per-URL progress:

```
● docs.python.org/3/tutorial/errors.html
◐ go.dev/blog/error-handling-and-go · fetching
◑ doc.rust-lang.org/book/ch09-00-error-... · extracting
```

Each URL independently transitions through: pending → fetching → extracting → summarizing → done/error. The browser pool manages concurrency automatically (6 tabs max).

## Extensions

pi-web-fetch has a hook system for site-specific fetch behavior. Extensions can intercept any stage of the pipeline: before fetch, after fetch (HTML), after extraction (markdown), or at summarization time.

### Built-in extensions

- **GitHub redirect** — matches `github.com/**`, redirects to `gh` CLI with context-aware suggestions (e.g. `gh issue view 123`)
- **Google Docs redirect** — matches `docs.google.com/**`, redirects to Google Workspace MCP tools

### Writing extensions

Extensions are TypeScript modules with a factory function default export:

```typescript
import type { WebFetchExtension } from "@kessokunet/pi-web-fetch/types";

export default function (): WebFetchExtension {
  return {
    name: "my-handler",
    matches: ["example.com/**"],
    async beforeFetch(ctx) {
      // Return a HookResult to short-circuit, or void to continue
      return ctx.redirect("Use a different tool for this site.");
    },
    async afterFetch(ctx) {
      // ctx.html — replace HTML or short-circuit
    },
    async afterExtract(ctx) {
      // ctx.markdown — replace markdown or short-circuit
    },
    async summarize(ctx) {
      // Override default LLM summarization
    },
  };
}
```

### Extension sources (in priority order)

1. **Event bus** — other pi extensions can register handlers via `pi.events.emit("web-fetch:register", extension)`
2. **Local** — TypeScript/JS files in `~/.pi/extensions/web-fetch/` (or configured `extensionsDir`)
3. **Built-in** — shipped with pi-web-fetch in the `extensions/` directory

## Configuration

Create `~/.pi/agent/web-fetch.json` to override defaults:

```json
{
  "model": "provider/model-id",
  "thinkingLevel": "medium",
  "extensionsDir": "~/.pi/extensions/web-fetch",
  "browserExecutable": "/usr/bin/chromium"
}
```

| Key | Default | Description |
|-----|---------|-------------|
| `model` | Current session model | Model for LLM content processing |
| `thinkingLevel` | Current session thinking level | Thinking level for the sub-agent |
| `extensionsDir` | `~/.pi/extensions/web-fetch/` | Directory for local extensions |
| `browserExecutable` | `PUPPETEER_EXECUTABLE_PATH`, then the first system browser found | Chromium-based browser to drive |

Without a config file, the extension uses whatever model and thinking level the current session is using.

## Architecture

```
web_fetch(url, prompt?)
  │
  ├─ URL validation & normalization (http→https, scheme check)
  ├─ Cache check (15-min TTL)
  ├─ Extension: beforeFetch hook
  ├─ Browser pool → Puppeteer page (networkidle2, 30s timeout)
  ├─ Cross-host redirect detection
  ├─ Extension: afterFetch hook
  ├─ trafilatura extraction (HTML → clean markdown)
  ├─ Extension: afterExtract hook
  ├─ Cache store
  └─ Content processing:
      ├─ With prompt → pi sub-agent (focused extraction)
      ├─ Small content, no prompt → return raw markdown
      └─ Large content, no prompt → pi sub-agent (structured summary)
```

For batch mode, each URL runs through this pipeline independently with its own browser tab. The browser pool (6 tabs max, 60s idle timeout) provides backpressure.

## License

MIT
