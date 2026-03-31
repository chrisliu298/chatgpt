---
name: chatgpt
effort: medium
description: |
  Send prompts to ChatGPT via Chrome browser automation and collect responses. Use this skill
  whenever the user wants to ask ChatGPT something, delegate a task to ChatGPT, get ChatGPT's
  perspective, or use ChatGPT's reasoning capabilities. Triggers on "ask chatgpt", "send to
  chatgpt", "use chatgpt", "chatgpt", "have chatgpt", or "chatgpt". Accepts two arguments:
  <model> <effort> where model is instant/thinking/pro and effort is low/medium/high/xhigh.
  Example: chatgpt thinking xhigh, chatgpt pro high, chatgpt instant.
allowed-tools: Bash(pbpaste), Write, mcp__claude-in-chrome__*
user-invocable: true
---

# ChatGPT

Send a prompt to ChatGPT via Chrome browser automation and save the response to a markdown file.

This is a convenience wrapper around the ChatGPT web UI, not a robust automation API. It depends on your logged-in Chrome session, the current DOM structure, clipboard state, and the claude-in-chrome MCP extension being connected.

## Prerequisites

- The claude-in-chrome MCP extension must be installed and connected to your Claude Code session.
- ChatGPT must be logged in within Chrome.
- Primary tested environment is macOS. Clipboard extraction uses `pbpaste`. On Linux, use `xclip -selection clipboard -o` or `wl-paste`. On Windows, use `powershell -NoProfile -Command Get-Clipboard`.

## Arguments

```text
chatgpt <model> [effort]
```

**Models:** `instant`, `thinking`, `pro`

**Effort levels:**

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

Note: Pro's UI default is Extended (`high`), so selecting `pro` without explicit effort requires changing the effort pill from Extended to Standard.

**Typical response times** (vary widely with prompt complexity, output length, and server load — treat as rough guidance):

| Model + effort | Typical wait |
|---------------|-------------|
| `instant` | 1–5 seconds |
| `thinking low` | 2–10 seconds |
| `thinking medium` | 5–30 seconds |
| `thinking high` | 20 seconds – 2 minutes |
| `thinking xhigh` | 30 seconds – 5 minutes |
| `pro medium` | 20–60 seconds (short), 2–10 minutes (long) |
| `pro high` | 1–10 minutes typical, 15–30+ minutes for long outputs |

## Limitations

- **Text-only prompts.** File attachments and images are not supported. If the user needs to send an image or file, inform them this requires manual interaction.
- **Text-only responses.** The copy button captures markdown text only. Images, canvases, and other non-text artifacts are not captured. If the response includes non-text content, take a screenshot and tell the user.
- **macOS clipboard.** The primary extraction method uses `pbpaste`. See Prerequisites for cross-platform alternatives.

## Workflow

All browser interactions use `mcp__claude-in-chrome__computer` actions (e.g., `left_click`, `type`, `key`, `wait`, `screenshot`, `scroll_to`, `zoom`). Page inspection uses `mcp__claude-in-chrome__read_page`. JavaScript execution uses `mcp__claude-in-chrome__javascript_tool`.

### 1. Open ChatGPT

Navigate to `https://chatgpt.com`. After the page loads, call `read_page` with `filter: "interactive"` to confirm the textbox and model selector button are present.

If they are missing after two retries, take a screenshot — the page may be showing a login screen, CAPTCHA, or error. Report to the user if unrecoverable.

### 2. Start a new chat

If the URL contains `/c/` (an existing conversation) or any assistant messages are visible, click "New chat" in the sidebar first.

Call `read_page` again and confirm the main prompt textbox is present.

If the user explicitly wants to continue the current conversation, skip this step and keep the current chat and model.

### 3. Select the model

1. Click the button with accessibility label "Model selector" (top of page).
2. The dropdown shows menu items that are **unlabeled in the accessibility tree**. Their order from top to bottom is: Instant (1st), Thinking (2nd), Pro (3rd), Configure (4th). Take a screenshot to verify the order for the current account, then click the item at the correct position.
3. If the requested model is not present in the menu, stop and report which models are available.

Do not assume the options are in a fixed order. Use the current visible text or take a screenshot to verify before clicking by position.

For `instant`, skip effort selection and continue to step 5.

### 4. Set the effort level

Only for `thinking` and `pro`.

1. Call `read_page` with `filter: "interactive"`. Near the bottom of the input area, find a button whose text contains the model name — this is the effort pill.
2. Click the effort pill to open the effort dropdown.
3. Click the UI label that matches the requested effort (see the mapping table in Arguments).
4. Verify by calling `read_page` again and confirming the effort pill text matches the expected label.

Expected pill labels after selection:
- Thinking: "Light", "Thinking" (Standard), "Extended thinking", or "Heavy thinking"
- Pro: "Pro" (Standard) or "Extended Pro"

If the UI does not expose the expected option, stop and report what is visible.

### 5. Send the prompt

Before sending, record the current assistant message count — this is needed for accurate completion detection:

```javascript
window.__chatgptAssistantCountBefore = document.querySelectorAll('[data-message-author-role="assistant"]').length;
```

Then:

1. Click the textbox (placeholder "Ask anything").
2. Type the prompt using the `computer` tool's `type` action.
3. Press Enter to submit using the `computer` tool's `key` action with text `Return`.

**Multi-line prompts:** The textbox sends on Enter. For prompts containing newlines, use `form_input` to set the full value at once (avoids premature submission), or use clipboard paste (`pbcopy` + Cmd+V keystroke).

**Long prompts (>2000 characters):** Break the text into chunks and type each chunk separately. The `type` action may drop characters on very long strings. After populating the textbox, consider taking a screenshot to spot-check the content before submitting.

Take a screenshot to confirm the prompt appears as a user message in the conversation. If the textbox still contains the prompt (submission failed), press Enter again.

### 6. Wait for completion

Higher effort levels think deeply and responses take a long time. This is expected, not a failure.

Poll by running this JS via `javascript_tool`:

```javascript
const msgs = [...document.querySelectorAll('[data-message-author-role="assistant"]')];
const lastMsg = msgs.at(-1);
const stop = document.querySelector('button[aria-label="Stop streaming"]')
  || document.querySelector('button[data-testid="stop-button"]')
  || document.querySelector('button[aria-label="Stop generating"]');
const actions = lastMsg
  ? lastMsg.parentElement?.querySelector('[role="group"][aria-label="Response actions"]')
  : null;
const done = msgs.length > (window.__chatgptAssistantCountBefore ?? 0) && !stop && !!actions;
JSON.stringify({ done, hasStopButton: !!stop, assistantCount: msgs.length, hasActionsOnLast: !!actions });
```

If `done` is false, the model is still generating. Take a screenshot periodically to visually confirm progress.

**Polling schedule** (max wait is a safety timeout — report to user if exceeded):

| Model + effort | First check | Poll interval | Max wait |
|---------------|------------|---------------|----------|
| `instant` | 5s | 5s | 1 min |
| `thinking low` | 5s | 5s | 1 min |
| `thinking medium` | 10s | 10s | 2 min |
| `thinking high` | 15s | 15s | 10 min |
| `thinking xhigh` | 30s | 30s | 20 min |
| `pro medium` | 30s | 30s | 15 min |
| `pro high` | 30s | 30s | 2 hours |

Between polls, use the `computer` tool's `wait` action. The `wait` action caps at 30 seconds — for longer intervals, chain multiple waits.

If max wait is exceeded, take a screenshot and inspect the page. Check for an error message, a completed response below the viewport (try scrolling down), or a still-generating indicator. Report to the user and ask how to proceed.

### 7. Extract the response

Extract the response using the "Copy response" button with clipboard validation:

1. **Read the clipboard before copy** — run `pbpaste` and store the result as `before`.
2. **Locate the copy button** — use JS to scope to the latest response:
   ```javascript
   const msgs = [...document.querySelectorAll('[data-message-author-role="assistant"]')];
   const lastMsg = msgs.at(-1);
   const actionsGroup = lastMsg?.parentElement?.querySelector('[role="group"][aria-label="Response actions"]');
   JSON.stringify({ found: !!actionsGroup });
   ```
   Then call `read_page` and click the "Copy response" button within that actions group. If the button is not visible, scroll to the bottom of the page first.
3. **Wait 1–2 seconds** for the clipboard to populate.
4. **Read the clipboard** — run `pbpaste` again as `after`.
5. **Validate** — if `after` is empty or identical to `before`, the copy failed. Click "Copy response" again, wait 2 seconds, and retry once. If still unchanged, report extraction failure to the user.

Always use the "Copy response" button — it preserves markdown formatting. Do not use `.innerText` for saved output.

### 8. Save the response

Write the clipboard content to a markdown file. Use a short, descriptive, kebab-case filename with a timestamp:

Example: `chatgpt-20260322-transformer-scaling.md`

Report the saved file path, the requested model and effort, and a brief summary of the response to the user.

## Multi-turn conversations

When the user wants to send a follow-up in the same ChatGPT conversation:

- Do not start a new chat or change the model.
- Record a fresh assistant count.
- Type the follow-up in the textbox and press Enter.
- Wait for completion and extract the new response using steps 6–8.

## Troubleshooting

- **Page not loaded**: Wait 3 seconds, call `read_page` again.
- **Model selector not found**: The page may not have fully loaded. Take a screenshot and retry.
- **Menu items unlabeled in accessibility tree**: Take a screenshot to verify the visible order before clicking by position. After clicking, verify the selected model by checking the effort pill text.
- **Effort pill not visible**: Only appears for `thinking` and `pro`. For `instant`, there is no effort selector — this is correct. For `thinking`/`pro`, wait 2 seconds after model selection, then `read_page` again.
- **Requested model not available**: The `pro` option is only available on accounts that expose it. Report to user.
- **Expected effort not available**: Report the visible effort options and stop.
- **Response seems stuck**: Take a screenshot. If the UI shows an error or rate limit message, report it. Long waits are normal for high-effort models.
- **"Something went wrong" during generation**: Look for a "Regenerate" button. If found, click it to retry, then restart polling from step 6.
- **Rate limit or usage cap**: ChatGPT may show a banner about reaching a usage cap. Report the exact message to the user and ask whether to retry after a delay.
- **Login screen, CAPTCHA, or interstitial**: Stop and tell the user what manual action is required in Chrome. Do not attempt to solve CAPTCHAs.
- **Copy button not found**: Scroll to the bottom of the page first — the response actions are at the end of the last message.
- **Clipboard unchanged after copy**: Click "Copy response" again, wait 2 seconds, retry. If still unchanged, report failure.
- **Canvas or side panel content**: Take a screenshot, inform the user, and suggest they extract canvas content manually.
