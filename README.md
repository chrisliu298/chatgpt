# ChatGPT

**A skill for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) that lets your agent send prompts to ChatGPT via Chrome and collect the responses.**

> *You type `/chatgpt thinking xhigh`, provide the prompt, and Claude drives the ChatGPT UI: selects the model, polls until the response finishes, copies the markdown, and saves it to a file.*

Uses the [Claude in Chrome](https://chromewebstore.google.com/detail/claude-in-chrome/oikdahcampanlblkmhfaigiheadcmfkb) extension for browser automation — no API keys, no headless browser, just your logged-in ChatGPT session. This is a convenience wrapper, not a robust automation API.

Invoke with `/chatgpt <model> [effort]` or ask your agent to "send this to chatgpt", "ask chatgpt", or "use chatgpt".

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

1. **Claude Code** — installed and running ([docs](https://docs.anthropic.com/en/docs/claude-code))
2. **Claude in Chrome extension** — install from the [Chrome Web Store](https://chromewebstore.google.com/detail/claude-in-chrome/oikdahcampanlblkmhfaigiheadcmfkb), then connect it to your Claude Code session
3. **ChatGPT account** — logged in within Chrome. The `pro` model is only available on accounts where it appears in the ChatGPT model picker
4. **Clipboard permission** — when prompted by Chrome, allow clipboard access for chatgpt.com (needed to copy responses)

### Connecting Claude in Chrome

1. Install the extension from the Chrome Web Store link above
2. Open Chrome and click the extension icon — it should show "Connected" when Claude Code is running
3. Verify by asking Claude Code to take a screenshot of your current tab

---

## How It Works

```
┌──────────────────────────────────────────────┐
│  1. Navigate to chatgpt.com (fresh chat)     │
│  2. Select model + effort level              │
│  3. Type prompt, verify submission           │
│  4. Poll for completion (seconds to minutes) │
│  5. Click "Copy response" + validate clip    │
│  6. Save markdown to file                    │
└──────────────────────────────────────────────┘
```

The skill automates the full ChatGPT web UI workflow through Chrome — clicking buttons, selecting dropdowns, typing text, and reading responses — exactly as a human would.

---

## Installation

Clone into your Claude Code skills directory:

```bash
git clone https://github.com/chrisliu298/chatgpt.git ~/.claude/skills/chatgpt
```

---

## Usage

```
/chatgpt <model> [effort]
```

**Examples:**

```
/chatgpt thinking xhigh
> Analyze the convergence properties of Adam vs SGD with momentum...

/chatgpt pro high
> Survey the literature on test-time compute scaling...

/chatgpt instant
> Summarize this paragraph in one sentence...
```

The agent will:

1. Open a new ChatGPT chat in Chrome
2. Select the requested model and effort level
3. Send your prompt
4. Wait for the response to complete (polling at appropriate intervals)
5. Copy the response with clipboard validation and save it as a markdown file

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

After the initial response, you can send follow-up messages in the same ChatGPT conversation. The agent reuses the existing chat without changing the model or starting over.

---

## Limitations

- **Text-only prompts** — file attachments and image uploads are not supported
- **Text-only responses** — images, canvases, and other non-text artifacts are not captured by the copy button
- **Browser-dependent** — requires a live Chrome session with the extension connected; not suitable for headless or unattended environments
- **macOS primary** — clipboard extraction uses `pbpaste` by default; Linux and Windows users need alternative clipboard commands

---

## Contributors

- [@chrisliu298](https://github.com/chrisliu298)
- **Claude Code** — skill design and browser automation protocol
