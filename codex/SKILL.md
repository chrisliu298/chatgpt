---
name: chatgpt
description: |
  Send prompts to ChatGPT via chrome-devtools-mcp and collect responses. Use this skill
  whenever the user wants to ask ChatGPT something, delegate a task to ChatGPT, get ChatGPT's
  perspective, or use ChatGPT's reasoning capabilities. Triggers on "ask chatgpt", "send to
  chatgpt", "use chatgpt", "chatgpt", "have chatgpt", or $chatgpt. Accepts two arguments:
  <model> <effort> where model is instant/thinking/pro and effort is low/medium/high/xhigh.
  Example: $chatgpt thinking xhigh, $chatgpt pro high, $chatgpt instant.
user-invocable: true
---

# ChatGPT

Send a prompt to ChatGPT via Chrome browser automation and save the response to a markdown file.

This is a convenience wrapper around the ChatGPT web UI, not a robust automation API. It depends on a live signed-in Chrome session, the current DOM structure, and the `chrome-devtools-mcp` server being available in Codex.

## Prerequisites

- `chrome-devtools-mcp` must be configured as a Codex MCP server and visible in the current session.
- Google Chrome must already be running locally with ChatGPT signed in.
- Chrome remote debugging must be enabled at `chrome://inspect/#remote-debugging` so `chrome-devtools-mcp --autoConnect` can attach.
- Use the live signed-in Chrome session. Do not switch to headless Chrome, an isolated profile, or a fresh unauthenticated browser.
- Primary tested environment is macOS. Clipboard verification uses `pbpaste`. On Linux, use `xclip -selection clipboard -o` or `wl-paste`. On Windows, use `powershell -NoProfile -Command Get-Clipboard`.

## Arguments

```text
$chatgpt <model> [effort]
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
- **Text-only responses.** DOM text extraction may lose markdown formatting. Images, canvases, and other non-text artifacts are not captured. If the response includes non-text content, take a screenshot and tell the user.
- **macOS clipboard.** The optional clipboard extraction uses `pbpaste`. See Prerequisites for cross-platform alternatives.
- **ChatGPT UI labels can drift.** Always inspect the live page before clicking controls and prefer visible text over assumptions.

## Workflow

Use the Chrome DevTools MCP tools for all browser interactions, not shell commands. Take a fresh snapshot before every interaction that depends on page state.

### 1. Open ChatGPT

- Call `mcp__chrome_devtools__list_pages`.
- If a `chatgpt.com` tab is already present, select it with `mcp__chrome_devtools__select_page`.
- If not, open `https://chatgpt.com/` with `mcp__chrome_devtools__new_page`.
- Take a fresh `mcp__chrome_devtools__take_snapshot`.

If the page is showing a login screen, CAPTCHA, or error after two retries, stop and tell the user what manual step is required.

### 2. Start a new chat

If the current page is an existing conversation and the user did not ask to continue it:

- Take a fresh snapshot.
- Click the visible `New chat` control.
- Take another snapshot and confirm the main prompt textbox is present.

If the user explicitly wants to continue the current conversation, skip this step and keep the current chat and model.

### 3. Select the model

- Take a fresh snapshot.
- Find the visible model control near the top or input area.
- If the currently visible model already matches the requested base model, keep it.
- Otherwise click the model selector and choose the requested visible option by text.

Do not assume the options are in a fixed order. Use the current visible text from the live snapshot.

If the requested model is not shown in the current UI, stop and report which model controls or options are visible.

For `instant`, skip effort selection and continue to step 5.

### 4. Set the effort level

Only for `thinking` and `pro`.

- Take a fresh snapshot.
- Find the visible current-model chip or effort control near the input area.
- Click it to open the effort menu.
- Select the visible option that matches the requested effort label (see the mapping table in Arguments).
- Take another snapshot and verify the selected chip text.

Expected pill labels after selection:
- Thinking: "Light", "Thinking" (Standard), "Extended thinking", or "Heavy thinking"
- Pro: "Pro" (Standard) or "Extended Pro"

If the UI does not expose the expected option, stop and report what is visible.

### 5. Send the prompt

Before sending, record the current assistant message count — this is needed for accurate completion detection:

```javascript
() => {
  window.__chatgptAssistantCountBefore =
    document.querySelectorAll('[data-message-author-role="assistant"]').length;
  return "ok";
}
```

Then:

- Take a fresh snapshot.
- Find the main prompt textbox.
- Use `mcp__chrome_devtools__fill` for the full prompt text.
- Submit with the visible `Send prompt` button when available. If needed, use `mcp__chrome_devtools__press_key` with `Enter`.

Take a snapshot to confirm the prompt appeared in the conversation. If the textbox still contains the prompt (submission failed), press Enter again.

### 6. Wait for completion

Higher effort levels think deeply and responses take a long time. This is expected, not a failure.

Poll with `mcp__chrome_devtools__evaluate_script`:

```javascript
() => {
  const msgs = [...document.querySelectorAll('[data-message-author-role="assistant"]')];
  const lastMsg = msgs.at(-1);
  const stop =
    document.querySelector('button[aria-label="Stop streaming"]') ||
    document.querySelector('button[data-testid="stop-button"]') ||
    document.querySelector('button[aria-label="Stop generating"]');
  const actions = lastMsg
    ? lastMsg.parentElement?.querySelector('[role="group"][aria-label="Response actions"]')
    : null;
  return {
    done:
      msgs.length > (window.__chatgptAssistantCountBefore ?? 0) &&
      !stop &&
      !!actions,
    assistantCount: msgs.length,
    hasStopButton: !!stop,
    hasActionsOnLast: !!actions,
  };
}
```

If `done` is false, the model is still generating. Take a snapshot periodically to visually confirm progress.

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

Between polls, use `mcp__chrome_devtools__wait_for` or short shell sleeps.

If max wait is exceeded, take a snapshot and inspect the page. Check for an error message, a completed response below the viewport (try scrolling down), or a still-generating indicator. Report to the user and ask how to proceed.

### 7. Extract the response

Primary path: extract the latest assistant message text from the DOM with `mcp__chrome_devtools__evaluate_script`:

```javascript
() => {
  const msgs = [...document.querySelectorAll('[data-message-author-role="assistant"]')];
  const lastMsg = msgs.at(-1);
  return lastMsg ? lastMsg.innerText : "";
}
```

Use that returned text as the saved output. If the extracted text is empty, stop and report the extraction failure.

**Optional clipboard path** — only try this if the user explicitly wants to preserve markdown formatting:

1. Read the current clipboard with `pbpaste` and store as `before`.
2. Click the visible `Copy response` button for the latest response. If the button is not visible, scroll to the bottom of the page first.
3. Wait 1–2 seconds, then read `pbpaste` again as `after`.
4. If `after` is empty or identical to `before`, click `Copy response` once more, wait 2 seconds, and check again.
5. If the clipboard still does not change, fall back to DOM text extraction and tell the user the saved output may lose markdown formatting.

### 8. Save the response

Write the extracted response text to a markdown file. Use a short, descriptive, kebab-case filename with a timestamp:

Example: `chatgpt-20260322-transformer-scaling.md`

Report the saved file path, the requested model and effort, whether the output came from DOM extraction or clipboard copy, and a brief summary of the response to the user.

## Multi-turn conversations

When the user wants to send a follow-up in the same ChatGPT conversation:

- Do not start a new chat or change the model.
- Record a fresh assistant count.
- Type the follow-up and submit.
- Wait for completion and extract the new response using steps 6–8.

## Troubleshooting

- **`chrome-devtools-mcp` unavailable**: Stop and tell the user the Codex session needs the MCP server configured. Do not continue with another browser backend.
- **Live Chrome session not attached**: Stop and report. Tell the user to open Chrome, keep ChatGPT signed in, and enable remote debugging.
- **Page not loaded**: Take a snapshot and retry after a few seconds.
- **Model selector not found**: The page may not have fully loaded. Take a snapshot and retry.
- **Effort pill not visible**: Only appears for `thinking` and `pro`. For `instant`, there is no effort selector — this is correct. For `thinking`/`pro`, wait 2 seconds after model selection, then take a snapshot again.
- **Requested model not available**: The `pro` option is only available on accounts that expose it. Report to user.
- **Expected effort not available**: Report the visible effort options and stop.
- **Response seems stuck**: Take a snapshot. If the UI shows an error or rate limit message, report it. Long waits are normal for high-effort models.
- **"Something went wrong" during generation**: Look for a "Regenerate" button. If found, click it to retry, then restart polling from step 6.
- **Rate limit or usage cap**: ChatGPT may show a banner about reaching a usage cap. Report the exact message to the user and ask whether to retry after a delay.
- **Login screen, CAPTCHA, or interstitial**: Stop and tell the user what manual action is required in Chrome. Do not attempt to solve CAPTCHAs.
- **Copy button not found**: Use DOM text extraction instead.
- **Clipboard unchanged after copy**: Retry once, then use DOM text extraction and warn that markdown formatting may be lost.
- **Canvas or side panel content**: Take a screenshot, inform the user, and suggest they extract canvas content manually.
