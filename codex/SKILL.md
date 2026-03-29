---
name: chatgpt
effort: medium
description: |
  Send prompts to ChatGPT via browser-use and collect responses. Use this skill
  whenever the user wants to ask ChatGPT something, delegate a task to ChatGPT, get ChatGPT's
  perspective, or use ChatGPT's reasoning capabilities. Triggers on "ask chatgpt", "send to
  chatgpt", "use chatgpt", "chatgpt", "have chatgpt", or $chatgpt. Accepts two arguments:
  <model> <effort> where model is instant/thinking/pro and effort is low/medium/high/xhigh.
  Example: $chatgpt thinking xhigh, $chatgpt pro high, $chatgpt instant.
user-invocable: true
---

# ChatGPT

Send a prompt to ChatGPT through the ChatGPT web UI using `browser-use`, then save the response text to a file.

This is a convenience wrapper around the ChatGPT website, not an API client. It depends on your local Chrome profile, the current DOM structure, clipboard state, and `browser-use` being able to control a headed browser.

## Prerequisites

- `browser-use` must be installed and available on `PATH`. If `command -v browser-use` fails, stop and tell the user to install it before continuing.
- ChatGPT must already be logged in in the local Chrome profile `Default`.
- Use the real Chrome profile, not headless Chromium, not an empty browser, and not a fallback unauthenticated session.
- Primary tested environment is macOS. Clipboard extraction uses `pbpaste`.

## Arguments

```
$chatgpt <model> [effort]
```

**Models:** `instant`, `thinking`, `pro`

**Effort levels** (consistent with codex effort conventions):

| Argument | Thinking UI label | Pro UI label |
|----------|------------------|--------------|
| `low` | Light | — |
| `medium` | Standard | Standard |
| `high` | Extended | Extended |
| `xhigh` | Heavy | — |

- `instant` takes no effort argument.
- `thinking` accepts `low`, `medium`, `high`, `xhigh`.
- `pro` accepts `medium` and `high`.
- If the model is `thinking` or `pro` and no effort is supplied, default to `medium`.
- If the model is `instant`, do not apply an effort.
- If the model is omitted, ask the user which model they want.

## Limits

- Text-only prompts. If the user wants attachments or images, tell them this skill does not support that flow.
- Text-only response capture. The reliable Codex path is DOM text extraction from the latest assistant message. This may lose markdown formatting.
- Treat the ChatGPT `Copy response` button as best effort only. Under browser automation, ChatGPT may show `Failed to copy to clipboard.` even when the click succeeds.
- No silent fallback. If the real Chrome profile cannot be opened or reused, stop and report the issue instead of switching to a fresh browser.

## Command Prefix

Use a named persistent session so follow-ups reuse the same browser state:

```bash
browser-use --session chatgpt --headed --profile "Default" ...
```

Do not drop `--profile "Default"`. Do not replace it with `--connect`, `--cdp-url`, or a headless session unless the user explicitly asks for a different mode.

## Workflow

### 1. Verify `browser-use`

Run:

```bash
command -v browser-use
```

If missing, stop and tell the user `browser-use` is not installed. Mention the official installer:

```bash
curl -fsSL https://browser-use.com/cli/install.sh | bash
```

After install, the user should verify with:

```bash
browser-use doctor
```

### 2. Open ChatGPT in the logged-in Chrome profile

Use:

```bash
browser-use --session chatgpt --headed --profile "Default" open https://chatgpt.com
```

Then inspect the page with:

```bash
browser-use --session chatgpt state
```

Use `state` before every interaction that depends on element indices. Resolve indices from visible text at that moment. Never hardcode index numbers from prior runs.

If the page is clearly a login screen, CAPTCHA, profile-lock error, or some other interstitial, stop and tell the user what manual step is required.

### 3. Start a new chat when needed

If the current URL already points at an existing conversation or visible assistant messages show this is not a blank chat:

- Use `state` to find the visible `New chat` control.
- Click it with `browser-use --session chatgpt click <index>`.
- Run `state` again to confirm the textbox and model selector are present.

If the user explicitly wants to continue the current conversation, skip this step and keep the current chat/model.

### 4. Select the requested model

Inspect the current prompt controls with `state`.

- Run `state`.
- The current UI exposes the active model as a visible chip or button, for example `Thinking`.
- If the visible chip already matches the requested base model, keep it and move on to effort selection.
- If a separate prompt-time model chooser is visible in `state`, click it and select the requested visible model option by text.

Do not assume the options are in a fixed order. Use current visible text from `state`.

If the requested model is not shown in the current prompt controls, stop and report which model controls/options are visible instead of guessing.

For `instant`, skip effort selection and continue to prompt entry.

### 5. Select the requested effort for `thinking` or `pro`

After model selection:

- Run `state`.
- Find the visible current-model chip near the input area. On the current UI, clicking `Thinking` opens the `Thinking effort` menu.
- Click the chip, run `state` again, and select the visible option that matches the requested effort label.

Expected mappings:

- `thinking low` -> `Light`
- `thinking medium` -> `Standard`
- `thinking high` -> `Extended`
- `thinking xhigh` -> `Heavy`
- `pro medium` -> `Standard`
- `pro high` -> `Extended`

Verify the selected pill text after changing it. If the UI does not expose the expected option, stop and report what is visible.

### 6. Enter the prompt and submit

Before sending, record the current assistant message count with `browser-use eval`:

```bash
browser-use --session chatgpt eval "window.__chatgptAssistantCountBefore = document.querySelectorAll('[data-message-author-role=\"assistant\"]').length; 'ok'"
```

Then:

- Run `state`.
- Find the main prompt textbox, usually labeled `Ask anything`.
- For short single-line prompts, use `browser-use --session chatgpt input <index> \"...\"`.
- For longer or multi-line prompts, click the textbox, then use `browser-use --session chatgpt type \"...\"` only if that will not submit prematurely. If the prompt contains multiple lines or is long enough that typing may be lossy, prefer focusing the textbox and pasting through the clipboard instead of splitting the text across many keystroke calls.
- Submit with:

```bash
browser-use --session chatgpt keys "Enter"
```

Run `state` again to confirm the prompt left the textbox and appeared in the conversation. If it is still in the textbox, press Enter once more.

### 7. Wait for completion

Use `browser-use eval` for completion detection and shell `sleep` between polls.

Poll command:

```bash
browser-use --session chatgpt eval "const msgs=[...document.querySelectorAll('[data-message-author-role=\"assistant\"]')]; const lastMsg=msgs.at(-1); const stop=document.querySelector('button[aria-label=\"Stop streaming\"]') || document.querySelector('button[data-testid=\"stop-button\"]') || document.querySelector('button[aria-label=\"Stop generating\"]'); const actions=lastMsg ? lastMsg.parentElement?.querySelector('[role=\"group\"][aria-label=\"Response actions\"]') : null; JSON.stringify({done: msgs.length > (window.__chatgptAssistantCountBefore ?? 0) && !stop && !!actions, assistantCount: msgs.length, hasStopButton: !!stop, hasActionsOnLast: !!actions});"
```

Polling schedule:

| Model + effort | First sleep | Interval | Max wait |
|---------------|-------------|----------|----------|
| `instant` | 5s | 5s | 1 min |
| `thinking low` | 5s | 5s | 1 min |
| `thinking medium` | 10s | 10s | 2 min |
| `thinking high` | 15s | 15s | 10 min |
| `thinking xhigh` | 30s | 30s | 20 min |
| `pro medium` | 30s | 30s | 15 min |
| `pro high` | 30s | 30s | 2 hours |

Between polls, use shell `sleep`. Do not assume a browser-use sleep subcommand exists.

If completion does not arrive before the max wait:

- run `state`
- take a screenshot with `browser-use --session chatgpt screenshot`
- inspect for an error banner, rate limit, or stuck generation state
- report the issue to the user

### 8. Extract the latest response

Primary path: extract the latest assistant message text from the DOM.

Use:

```bash
browser-use --session chatgpt eval '(() => { const msgs=[...document.querySelectorAll("[data-message-author-role=\"assistant\"]")]; const lastMsg=msgs.at(-1); return lastMsg ? lastMsg.innerText : ""; })()'
```

Use that returned text as the default saved output for Codex.

If the extracted text is empty, stop and report the extraction failure.

Optional best-effort clipboard path: only try this if the user explicitly wants to preserve markdown formatting and you are willing to accept that it may fail.

Read the current clipboard first:

```bash
pbpaste
```

Confirm the latest assistant message has response actions with:

```bash
browser-use --session chatgpt eval "const msgs=[...document.querySelectorAll('[data-message-author-role=\"assistant\"]')]; const lastMsg=msgs.at(-1); const actions=lastMsg?.parentElement?.querySelector('[role=\"group\"][aria-label=\"Response actions\"]'); JSON.stringify({found: !!actions});"
```

Then:

- scroll if needed
- run `state`
- find the visible `Copy response` button associated with the latest response
- click it
- wait briefly with `sleep 2`
- run `pbpaste` again

If the clipboard is empty or unchanged, click `Copy response` once more, `sleep 2`, and check again.

If the clipboard still does not change, fall back to DOM text extraction for the latest assistant message:

```bash
browser-use --session chatgpt eval '(() => { const msgs=[...document.querySelectorAll("[data-message-author-role=\"assistant\"]")]; const lastMsg=msgs.at(-1); return lastMsg ? lastMsg.innerText : ""; })()'
```

Save that fallback text and tell the user the copy button did not update the system clipboard, so the saved output may lose markdown formatting.

### 9. Save the response

Write the extracted response text to a timestamped markdown file with a short kebab-case topic slug:

```text
chatgpt-YYYYMMDD-topic.md
```

Use a descriptive but short filename. Avoid collisions with a timestamp.

Report:

- saved file path
- requested model/effort
- whether the saved output came from DOM extraction or successful clipboard copy
- a brief summary of the ChatGPT response

## Multi-turn Conversations

When the user wants to send a follow-up in the same ChatGPT conversation:

- reuse the existing `chatgpt` browser-use session
- do not open a fresh chat
- do not change the model unless the user asks
- record a fresh assistant count
- submit the follow-up
- poll again
- copy only the latest response

## Troubleshooting

- **`browser-use` missing**: Stop and tell the user to install it. Do not continue.
- **Chrome/profile launch failure**: Stop and report it. Do not switch to headless Chromium or an empty session.
- **Login screen or CAPTCHA**: Tell the user which manual action is required in Chrome.
- **Requested model missing**: Report which model options are visible.
- **Expected effort missing**: Report the visible effort options and stop.
- **Visible model chip opens only effort controls**: Treat that chip as the current model selector state. If the requested base model is not otherwise visible, report the visible model controls and stop.
- **Rate limit or usage cap**: Quote the visible message and ask the user how to proceed.
- **Copy button not visible**: This does not block Codex extraction. Use DOM text extraction instead.
- **Clipboard unchanged after copy**: Retry once, then use DOM text extraction and warn that markdown formatting may be lost.
- **Canvas or side panel content**: Take a screenshot and tell the user the saved output may omit non-text artifacts.
