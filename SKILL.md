---
description: |
  Send prompts to ChatGPT via Chrome browser automation and collect responses. Use this skill
  whenever the user wants to ask ChatGPT something, delegate a task to ChatGPT, get ChatGPT's
  perspective, or use ChatGPT's reasoning capabilities. Triggers on "ask chatgpt", "send to
  chatgpt", "use chatgpt", "chatgpt", "have chatgpt", or /chatgpt. Accepts two arguments:
  <model> <effort> where model is instant/thinking/pro and effort is low/medium/high/xhigh.
  Example: /chatgpt thinking xhigh, /chatgpt pro high, /chatgpt instant.
allowed-tools: Bash(pbpaste), Write, mcp__claude-in-chrome__*
user-invocable: true
---

# ChatGPT

Send a prompt to ChatGPT via Chrome browser automation and save the response to a markdown file.

This is a convenience wrapper around the ChatGPT web UI, not a robust automation API. Browser automation depends on your logged-in session, current DOM structure, clipboard state, and Chrome staying active.

**Prerequisites:** ChatGPT must be logged in within Chrome, and the claude-in-chrome MCP extension must be connected. Primary tested environment is macOS.

## Arguments

```
/chatgpt <model> [effort]
```

**Models:** `instant`, `thinking`, `pro`

**Effort levels** (consistent with codex effort conventions):

| Argument | Thinking UI label | Pro UI label |
|----------|------------------|--------------|
| `low` | Light | — |
| `medium` | Standard | Standard |
| `high` | Extended | Extended |
| `xhigh` | Heavy | — |

- `instant` takes no effort argument (it has no effort selector)
- `thinking` accepts all four levels: `low`, `medium`, `high`, `xhigh`
- `pro` accepts `medium` and `high` only (the Pro UI only has Standard and Extended)

If the user specifies `thinking` or `pro` without an effort, default to `medium`. If they specify `instant`, do not apply an effort argument. If the user omits the model entirely, ask which model they want. Note: Pro's UI default is Extended (`high`), so selecting `pro` without explicit effort requires changing the effort pill from Extended to Standard.

**Typical response times** (vary widely with prompt complexity, output length, and server load — treat as rough guidance):
- `instant` — 1–5 seconds
- `thinking low` — 2–10 seconds
- `thinking medium` — 5–30 seconds
- `thinking high` — 20 seconds – 2 minutes
- `thinking xhigh` — 30 seconds – 5 minutes
- `pro medium` — 20–60 seconds for short answers, 2–10 minutes for long outputs
- `pro high` — 1–10 minutes typical, 15–30+ minutes for research-grade tasks with long outputs

## Limitations

- **Text-only prompts.** File attachments and images are not supported by this skill. If the user needs to send an image or file, inform them this requires manual interaction.
- **Text-only responses.** The copy button captures markdown text only. Images, canvases, and other non-text artifacts are not captured. If the response includes non-text content, take a screenshot and tell the user those artifacts were not included in the saved file.
- **macOS clipboard.** The primary extraction method uses `pbpaste`. On Linux, use `xclip -selection clipboard -o` or `wl-paste`. On Windows, use `powershell -NoProfile -Command Get-Clipboard`.

## Workflow

All browser interactions use `mcp__claude-in-chrome__computer` actions (e.g., `left_click`, `type`, `key`, `wait`, `screenshot`, `scroll_to`, `zoom`). Page inspection uses `mcp__claude-in-chrome__read_page`. JavaScript execution uses `mcp__claude-in-chrome__javascript_tool`.

### 1. Start a new chat

Navigate to `https://chatgpt.com`. After the page loads, verify you are in a blank conversation:
- If the URL contains `/c/` (an existing conversation) or any assistant messages are visible, click "New chat" in the sidebar first.
- Wait 2–3 seconds, then call `read_page` with `filter: "interactive"` to confirm the textbox and model selector button are present.
- If they are missing after two retries, take a screenshot — the page may be showing a login screen, CAPTCHA, or error. Report to the user if unrecoverable.

### 2. Select the model

**Select the base model:**
1. Click the button with accessibility label "Model selector" (top of page)
2. The dropdown shows menu items that are **unlabeled in the accessibility tree**. Their order from top to bottom is: Instant (1st), Thinking (2nd), Pro (3rd), Configure (4th). Take a screenshot to verify the order for the current account, then click the item at the correct position.
3. If the requested model is not present in the menu, stop and report which models are available. Ask the user whether to continue with an available option.

For `instant`, skip the effort selection below and proceed to step 3.

**Set the effort level (thinking and pro only):**
1. After selecting the base model, call `read_page` with `filter: "interactive"`. Near the bottom of the input area, find a button whose text contains the model name ("Thinking", "Pro", "Extended Pro", "Heavy thinking", etc.) — this is the effort pill.
2. Click the effort pill to open the effort dropdown.
3. The dropdown header reads "Thinking effort" or "Pro thinking effort".
4. Click the UI label that matches the requested effort (see the mapping table above).

**Verify** by calling `read_page` again and confirming the effort pill text matches the expected label:
- Thinking: "Light", "Thinking" (Standard), "Extended thinking", or "Heavy thinking"
- Pro: "Pro" (Standard) or "Extended Pro"

### 3. Send the prompt

1. Click the textbox (placeholder "Ask anything")
2. Type the prompt using the `computer` tool's `type` action
3. Press Enter to submit using the `computer` tool's `key` action with text `Return`

**Multi-line prompts:** The textbox sends on Enter. For prompts containing newlines, use `form_input` to set the full value at once (avoids premature submission), or use clipboard paste (`pbcopy` + Cmd+V keystroke).

**Long prompts (>2000 characters):** Break the text into chunks and type each chunk separately. The `type` action may drop characters on very long strings. After populating the textbox, consider taking a screenshot to spot-check the content before submitting.

**Verify submission:** After pressing Enter, wait 2 seconds and take a screenshot to confirm the prompt appears as a user message in the conversation. If the textbox still contains the prompt (submission failed), press Enter again.

**Record the assistant message count** before the response starts generating — this is needed for accurate completion detection:
```javascript
window.__assistantCountBefore = document.querySelectorAll('[data-message-author-role="assistant"]').length;
```

### 4. Wait for completion

Higher effort levels think deeply and responses take a long time. This is expected, not a failure.

**Completion detection** — poll by running this JS via `javascript_tool`:

```javascript
const msgs = [...document.querySelectorAll('[data-message-author-role="assistant"]')];
const lastMsg = msgs.at(-1);
const stop = document.querySelector('button[aria-label="Stop streaming"]')
  || document.querySelector('button[data-testid="stop-button"]')
  || document.querySelector('button[aria-label="Stop generating"]');
const actions = lastMsg
  ? lastMsg.parentElement?.querySelector('[role="group"][aria-label="Response actions"]')
  : null;
const done = msgs.length > (window.__assistantCountBefore ?? 0) && !stop && !!actions;
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

**If max wait is exceeded:** Take a screenshot and inspect the page. Check for an error message, a completed response below the viewport (try scrolling down), or a still-generating indicator. Report to the user and ask how to proceed.

### 5. Extract the response

Once the latest response is complete, extract it using the "Copy response" button with clipboard validation:

1. **Read the clipboard before copy** — run `pbpaste` and store the result as `before` (for validation)
2. **Locate the last assistant message's copy button** — use JS to scope to the latest response:
   ```javascript
   const msgs = [...document.querySelectorAll('[data-message-author-role="assistant"]')];
   const lastMsg = msgs.at(-1);
   const actionsGroup = lastMsg?.parentElement?.querySelector('[role="group"][aria-label="Response actions"]');
   JSON.stringify({ found: !!actionsGroup });
   ```
   Then call `read_page` and click the "Copy response" button within that actions group. If the button is not visible, scroll to the bottom of the page first.
3. **Wait 1–2 seconds** for the clipboard to populate
4. **Read the clipboard** — run `pbpaste` again as `after`
5. **Validate** — if `after` is empty or identical to `before`, the copy failed. Click "Copy response" again, wait 2 seconds, and retry once. If still unchanged, report extraction failure to the user.

### 6. Save the response

Write the validated clipboard content to a markdown file. Use a short, descriptive, kebab-case filename with a timestamp to avoid collisions:

Example: `chatgpt-20260322-transformer-scaling.md`

Report the saved file path and a brief summary of the response to the user.

## Multi-turn conversations

If the user wants to send a follow-up in the same ChatGPT conversation:

1. Skip steps 1–2 (don't start a new chat or change the model)
2. Type the follow-up in the textbox and press Enter
3. Record the new assistant message count (step 3's JS snippet)
4. Wait for completion and extract the new response using the same steps 4–6

Always use the "Copy response" button on the latest message for extraction — it preserves markdown formatting. Do not use `.innerText` for saved output, as it strips all formatting.

## Troubleshooting

- **Model selector not found**: Page may not have loaded. Wait 3 seconds, call `read_page` again.
- **Effort pill not visible**: Only appears for `thinking` and `pro`. For `instant`, there is no effort selector — this is correct. For `thinking`/`pro`, wait 2 seconds after model selection, then `read_page` again.
- **Menu items unlabeled in accessibility tree**: Take a screenshot to verify the visible order before clicking by position. After clicking, verify the selected model by checking the effort pill text. If the menu differs from expected, report the observed options.
- **Requested model not available**: The `pro` option is only available on accounts that expose it. If missing, report to user.
- **Response seems stuck**: Take a screenshot. If the UI shows an error or rate limit message, report it. Long waits are normal for high-effort models.
- **"Something went wrong" during generation**: Look for a "Regenerate" button. If found, click it to retry, then restart polling from step 4.
- **Rate limit or usage cap**: ChatGPT may show a banner about reaching a usage cap. Report the exact message to the user and ask whether to retry after a delay.
- **Login screen, CAPTCHA, or interstitial**: Stop and tell the user what manual action is required in Chrome. Do not attempt to solve CAPTCHAs.
- **Copy button not found**: Scroll to the bottom of the page first — the response actions are at the end of the last message.
- **Clipboard unchanged after copy**: Click "Copy response" again, wait 2 seconds, retry. If still unchanged, report failure.
- **Canvas or side panel opened**: The copy button may not capture canvas content. Take a screenshot, inform the user, and suggest they extract canvas content manually.
