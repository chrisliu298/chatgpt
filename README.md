# ChatGPT

**A skill for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) and Codex that lets your agent send prompts to ChatGPT through your logged-in browser session and collect the responses.**

> *You type `/chatgpt thinking xhigh` in Claude Code or `$chatgpt thinking xhigh` in Codex, provide the prompt, and your agent drives the ChatGPT UI: selects the model, polls until the response finishes, captures the response, and saves it to a file.*

Claude Code uses the [Claude in Chrome](https://chromewebstore.google.com/detail/claude-in-chrome/oikdahcampanlblkmhfaigiheadcmfkb) extension. Codex uses `chrome-devtools-mcp` to attach to your live signed-in Chrome session. No API keys, no manual copy-paste, just your logged-in ChatGPT session. This is a convenience wrapper, not a robust automation API. The Codex variant treats DOM text extraction as the primary capture path, with ChatGPT's native copy button as an optional formatting-preserving path.

Invoke with `/chatgpt <model> [effort]` in Claude Code or `$chatgpt <model> [effort]` in Codex, or ask your agent to "send this to chatgpt", "ask chatgpt", or "use chatgpt".

## Table of Contents

- [Prerequisites](#prerequisites)
- [How It Works](#how-it-works)
- [Installation](#installation)
- [Usage](#usage)
- [Models and Effort Levels](#models-and-effort-levels)
- [Multi-turn Conversations](#multi-turn-conversations)
- [Limitations](#limitations)
- [Contributors](#contributors)

---

## Prerequisites

1. **Claude Code or Codex** — installed and running
2. **Agent-specific browser automation**
   - Claude Code: install the Claude in Chrome extension from the [Chrome Web Store](https://chromewebstore.google.com/detail/claude-in-chrome/oikdahcampanlblkmhfaigiheadcmfkb), then connect it to your Claude Code session
   - Codex: configure `chrome-devtools-mcp` as a Codex MCP server
3. **ChatGPT account** — logged in within Chrome. The `pro` model is only available on accounts where it appears in the ChatGPT model picker
4. **Live Chrome session** — Codex expects ChatGPT to already be open and signed in within your local Chrome session
5. **Clipboard permission** — when prompted by Chrome, allow clipboard access for chatgpt.com (needed to copy responses)

### Configuring `chrome-devtools-mcp` for Codex

```bash
npx -y chrome-devtools-mcp@latest --help
```

Then add it to Codex as an MCP server, for example:

```bash
codex mcp add chrome-devtools -- npx -y chrome-devtools-mcp@latest --autoConnect --channel stable --no-usage-statistics
```

Chrome must also have remote debugging enabled at `chrome://inspect/#remote-debugging`.

### Connecting Claude in Chrome

1. Install the extension from the Chrome Web Store link above
2. Open Chrome and click the extension icon — it should show "Connected" when Claude Code is running
3. Verify by asking Claude Code to take a screenshot of your current tab

### Chrome Session for Codex

Codex attaches to the live Chrome session via:

```bash
npx -y chrome-devtools-mcp@latest --autoConnect --channel stable
```

That means Chrome must already be open locally and signed into ChatGPT.

---

## How It Works

```
┌──────────────────────────────────────────────┐
│  1. Navigate to chatgpt.com (fresh chat)     │
│  2. Select model + effort level              │
│  3. Type prompt, verify submission           │
│  4. Poll for completion (seconds to minutes) │
│  5. Extract latest response text             │
│  6. Save response to file                    │
└──────────────────────────────────────────────┘
```

The skill automates the full ChatGPT web UI workflow through Chrome: clicking buttons, selecting dropdowns, typing text, and reading responses. The browser backend differs by agent, and so does the invocation syntax: `/chatgpt` in Claude Code, `$chatgpt` in Codex. Claude’s path can preserve markdown through ChatGPT’s copy button; Codex’s default path is plain-text extraction from the latest assistant message, with `Copy response` available when clipboard-preserving output is worth the extra step.

---

## Installation

If you use this dotfiles repo, `./dotfiles.sh` installs the correct agent-specific variant automatically.

For standalone installation, clone the repo somewhere neutral, then symlink the agent-specific subdirectory:

```bash
git clone https://github.com/chrisliu298/chatgpt.git ~/.cache/chatgpt-src
```

Claude Code:

```bash
ln -s ~/.cache/chatgpt-src/claude ~/.claude/skills/chatgpt
```

Codex:

```bash
ln -s ~/.cache/chatgpt-src/codex ~/.codex/skills/chatgpt
```

---

## Usage

Claude Code:

```text
/chatgpt <model> [effort]
```

Codex:

```text
$chatgpt <model> [effort]
```

**Examples:**

```text
/chatgpt thinking xhigh
> Analyze the convergence properties of Adam vs SGD with momentum...

$chatgpt pro high
> Survey the literature on test-time compute scaling...

$chatgpt instant
> Summarize this paragraph in one sentence...
```

The agent will:

1. Open a new ChatGPT chat in Chrome
2. Select the requested model and effort level
3. Send your prompt
4. Wait for the response to complete (polling at appropriate intervals)
5. Capture the response and save it as a markdown file

---

## Models and Effort Levels

### Models

| Model | Description |
|-------|-------------|
| `instant` | Fast responses, no thinking. Good for simple tasks |
| `thinking` | Chain-of-thought reasoning with configurable depth |
| `pro` | Research-grade intelligence for the hardest problems |

### Effort levels

Effort arguments are consistent with [Codex](https://github.com/openai/codex) conventions:

| Argument | Thinking UI | Pro UI |
|----------|------------|--------|
| `low` | Light | — |
| `medium` | Standard | Standard |
| `high` | Extended | Extended |
| `xhigh` | Heavy | — |

- `instant` — takes no effort argument
- `thinking` — accepts `low`, `medium`, `high`, `xhigh`
- `pro` — accepts `medium` and `high` only
- If effort is omitted, defaults to `medium`

### Typical response times

Vary widely with prompt complexity, output length, and server load. Treat as rough guidance.

| Combo | Typical wait |
|-------|-------------|
| `instant` | 1–5 seconds |
| `thinking low` | 2–10 seconds |
| `thinking medium` | 5–30 seconds |
| `thinking high` | 20 seconds – 2 minutes |
| `thinking xhigh` | 30 seconds – 5 minutes |
| `pro medium` | 20–60 seconds (short), 2–10 minutes (long) |
| `pro high` | 1–10 minutes typical, 15–30+ minutes for long outputs |

---

## Multi-turn Conversations

After the initial response, you can send follow-up messages in the same ChatGPT conversation. The agent reuses the existing chat without changing the model or starting over. In Codex, that reuse is anchored to the attached live ChatGPT tab.

---

## Limitations

- **Text-only prompts** — file attachments and image uploads are not supported
- **Text-only responses** — images, canvases, and other non-text artifacts are not captured
- **Codex formatting caveat** — the Codex variant saves DOM-extracted text by default, so markdown fidelity is not guaranteed under browser automation
- **Browser-dependent** — Claude requires the extension-connected Chrome session; Codex requires `chrome-devtools-mcp` to attach to a live signed-in Chrome session; not suitable for headless or unattended environments
- **macOS primary** — clipboard extraction uses `pbpaste` by default; Linux and Windows users need alternative clipboard commands

---

## Contributors

- [@chrisliu298](https://github.com/chrisliu298)
- **Claude Code** — original skill design and browser automation protocol
- **Codex** — `chrome-devtools-mcp` variant exposed as `$chatgpt`
