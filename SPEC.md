# SPEC: Live Traffic Dashboard for the Lab 4 Agent Proxy

Status: APPROVED (spec only; not implemented)   ·   Owner: Diana   ·   Date: 2026-11-03   ·   Rev 2

## 1. Purpose

`proxy.py` currently dumps raw JSON and server-sent event (SSE) chunks to a terminal,
which quickly gets hard to read. This dashboard shows the same traffic in a browser:
**every request the harness sends and every reply, as they happen, newest first**,
with each request split into its parts and every message available for drill-down.

### Goals
- Show every request and every reply live, newest first. Nothing is dropped or hidden from the list.
- Split every request into its parts: system instructions (in full), tool definitions,
  the conversation so far, and the new message.
- Show each tool call beside its result, even though the result arrives in a later request.
- Show the reply: the model's text and the tool calls it asks for.
- Provide statistics (tokens, cost, time, tool usage) alongside the messages, never instead of them.
- Make every message openable down to its raw JSON.

### Non-goals (v1)
- Changing, replaying or blocking traffic (the proxy stays a passive tap).
- Multiple users, remote access or logins.
- Long-term storage or analytics.
- Dark mode.

## 2. Context: what the traffic looks like

| Fact | Consequence for the dashboard |
|---|---|
| Each `POST /v1/messages` resends the **full** conversation history | Show it in full but collapsed, and highlight what is **new** since the previous request |
| A tool call appears in the reply to request N; its result is sent in request N+1 | Pair calls and results across requests by `tool_use_id` |
| Responses are SSE streams (`message_start`, `content_block_delta`, …) | Reassemble the deltas into whole text, thinking and tool_use blocks |
| Requests include a large `system` prompt and `tools` array | Show both in full, in their own collapsible sections, and flag them when they change |
| Several kinds of call share the connection (main agent, subagents, small-model utility calls, `count_tokens`) | Show all of them, labeled by kind, and allow filtering |
| Auth headers carry tokens | Redact them, and never show the full value |

## 3. Core concepts

- **Exchange**: one HTTP request and its reply. One card in the list.
- **Conversation**: a series of exchanges that share a message history (main agent, or one subagent).
- **Block**: one content item: `text`, `thinking`, `tool_use`, `tool_result`, `image`, or an error.
- **Tool pair**: one `tool_use` block (from a reply) and its `tool_result` block (from a later request), matched by `tool_use_id`.

## 4. Layout

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ Top bar: ● live | Pause | Search | Filters | Errors: 2 ⚠ | Totals: tokens · $ │
├──────────────┬───────────────────────────────────────────────────────────────┤
│ Conversations│  ▲ 3 new (shown when scrolled away from the top)              │
│  All      42 │  ┌─ Exchange #42 · 14:52:07 · Main · opus · 200 · 3.1 s ────┐ │
│  Main     30 │  │ [Request] [Reply] [Diff] [Raw]                            │ │
│  Subagent  8 │  │  ▸ SYSTEM        12,480 chars        (unchanged)          │ │
│  Utility   4 │  │  ▸ TOOLS         23 definitions      (unchanged)          │ │
│              │  │  ▸ CONVERSATION  18 earlier messages                      │ │
│ Stats panel  │  │  ▾ NEW MESSAGE   RESULT ◂ Bash (pairs with #41)           │ │
│  tokens/req  │  │  ▾ REPLY         CLAUDE text · TOOL ▸ Read · end: tool_use│ │
│  cost        │  └───────────────────────────────────────────────────────────┘ │
│  time        │  ┌─ Exchange #41 … (older, below) ───────────────────────────┐ │
│  tool usage  │                                                               │
└──────────────┴───────────────────────────────────────────────────────────────┘
```

- **Newest first**: new exchanges are inserted at the **top** of the list.
- Each exchange card appears as soon as its request arrives. Its Reply section fills in live as the stream arrives.
- Left sidebar: conversation lanes (with counts) and the stats panel (§7).

## 5. The exchange card

### 5.1 Header line
Exchange number, time, conversation label (Main / Subagent N / Utility), kind label
(`messages`, `count_tokens`, …), model, HTTP status, duration, time to first token,
input/output/cache tokens, estimated cost, and badges for errors and changes.

### 5.2 Request: four clearly separated sections
Each section has its own header bar, tint and collapse toggle, always in this order:

| Section | Content | Default |
|---|---|---|
| `SYSTEM` | The system instructions **in full** (every block, including cache-control markers), with character and token counts | Collapsed; badge "changed" if different from the previous request in this conversation |
| `TOOLS` | Every tool definition: name, description, and input schema as a JSON tree. Collapsed list of names first, each expandable | Collapsed; badge "+N / −N / changed" if different from the previous request |
| `CONVERSATION SO FAR` | Every earlier message in `messages`, each as a styled block (§6) | Collapsed, showing "N earlier messages" |
| `NEW MESSAGE` | Messages added since the previous request in this conversation (usually the user prompt or the tool results) | **Expanded** |

Other request fields (model, `max_tokens`, `thinking`, `temperature`, `metadata`, …) appear as a small key/value strip under the header.

### 5.3 Reply
- The model's text, thinking, and the tool calls it asks for, in stream order.
- `stop_reason` shown as a tag (`end_turn`, `tool_use`, `max_tokens`, …).
- `usage` (input, output, cache read, cache write) shown in the header.
- Expanded by default. Streams live with a typing cursor; marked done at `message_stop`.

### 5.4 Tabs on every card
- **Request / Reply**: the structured views above.
- **Diff**: what changed since the previous request in the same conversation: new messages,
  plus any change to system, tools, model or parameters, shown as a line diff.
- **Raw**: request headers (redacted), request body as a JSON tree, and the raw SSE event list.

## 6. Visual styling

### 6.1 Message kinds

Every block uses **colour + tag + position + icon**, so nothing depends on colour alone.
Colours are soft pastel backgrounds with a stronger left border and tag in the same hue.

| Kind | Tag | Background / accent | Position | Notes |
|---|---|---|---|---|
| User prompt (typed by a person) | `YOU` | Light blue `#E8F1FD` / `#2F6FD6` | Right-aligned bubble | Bold |
| Model response (text) | `CLAUDE` | White `#FFFFFF` with soft shadow / `#444` | Left-aligned, full width | Rendered as Markdown |
| Model thinking | `THINKING` | Very light grey `#F5F5F5` / `#888`, italic | Left, indented | Collapsed by default |
| Tool call (`tool_use`) | `TOOL ▸ <name>` | Light amber `#FFF4E0` / `#C77700` | Left half of a tool pair | Header line shows the key argument (e.g. `Bash: ls -la`); input as a JSON tree |
| Tool output (`tool_result`) | `RESULT ◂ <name>` | Light green `#EAF7EE` / `#2E8B57`, or light red `#FDECEC` / `#C62828` on `is_error` | Right half of a tool pair | Collapsed if long |
| Injected context (`<system-reminder>` etc.) | `CONTEXT` | Light grey `#F3F3F3` / `#999` | Left, small | Collapsed by default |
| Request sections | `SYSTEM` / `TOOLS` / `CONVERSATION` / `NEW MESSAGE` / `REPLY` | Section header bars: lavender `#F3EEFC`, slate `#EEF1F5`, grey `#F7F7F7`, blue-white `#F0F6FF`, white | Full-width bars inside the card | Distinct tint + bold label + divider so the parts are clearly apart |
| Errors (HTTP ≥ 400, upstream down, `is_error`) | `ERROR` | Light red `#FDECEC` / `#C62828` | Full-width banner; red edge on the whole card | Always expanded |

- `tool_result` messages technically have role `user`. They must **not** be styled as user prompts.
- Text injected by the harness also arrives as user text. It gets the CONTEXT style, not YOU.

### 6.2 Tool calls beside their results
- A tool call and its result are shown **side by side** in a two-column row: call on the left, result on the right.
- In the reply to request N, the row shows the call with a "waiting…" placeholder on the right.
  When request N+1 arrives with the matching `tool_result`, the placeholder is filled in.
- The same pair also appears in request N+1's NEW MESSAGE section. Each side links to the
  exchange it came from ("call from #41", "result in #42"). Clicking a link scrolls to that exchange.
- Several parallel calls in one reply give several pair rows, matched by `tool_use_id`, never by order.
- On narrow windows (< 900 px) the pair stacks vertically but stays in one bordered box.

### 6.3 Readable formatting
- **Long text collapsible**: any block over ~10 lines shows the first 5, then "… +N lines (expand)".
- **JSON readable**: all JSON (tool inputs, schemas, raw bodies) is pretty-printed with syntax
  highlighting, in a collapsible tree, with a copy button. Strings that themselves contain JSON can be expanded inline.
- **Parts clearly apart**: request sections are separated by tinted header bars and dividers (§6.1). Request and Reply are separated by a heavier rule.

### 6.4 Theme
- **Light theme only**: off-white page background (`#FAFAF7`), dark grey text (`#222`),
  thin light-grey dividers (`#E5E5E5`). No dark mode in v1.
- All text/background pairs meet WCAG AA contrast (accent colours are used for tags and borders, never for body text on a pastel background).
- Monospace font for tool calls, results and JSON; a readable proportional font for prose.

## 7. Statistics

Statistics sit **alongside** the messages. Every number links back to the messages behind it.

- **Per request** (card header): input, output, cache-read and cache-write tokens; estimated cost; duration; time to first token.
- **Totals panel** (sidebar, live), overall and per conversation:
  - Requests, total tokens by type, estimated cost.
  - Time: total, average and slowest request.
  - **Tool usage**: calls per tool, error count per tool, average result size.
    Clicking a tool name filters the list to that tool's pairs.
  - A small tokens-per-request sparkline. Clicking a bar jumps to that exchange.
- **Cost** uses a per-model price table kept in one place in the code (input, output, cache read, cache write per million tokens) and is always labeled "estimated". An unknown model shows "—" rather than a guess.

## 8. Usability

### 8.1 Live flow (newest first)
- **Pinned to top**: while the view is at the top, new exchanges appear there in view.
  When the user scrolls down to read older ones, the view stays where it is and a floating
  "▲ N new" pill appears. Clicking it jumps to the top and resumes following.
- **Pause** button: freezes the list while still buffering traffic in the background. Resume inserts the buffer.

### 8.2 Everything visible, noise manageable
- **Every** request is listed, including `count_tokens`, title/summary utility calls and other endpoints. Utility calls are labeled and visually dimmed, not hidden.
- Optional filter "Hide utility calls" (off by default).
- **Conversation lanes** in the sidebar: All / Main / Subagent N / Utility, with counts.
- Global "Collapse all" / "Expand all" toggles.

### 8.3 Finding things
- **Filter chips**: YOU / CLAUDE / THINKING / TOOL / RESULT / CONTEXT / ERROR.
- **Filter by tool**: a dropdown of all tool names seen; shows only exchanges that contain that tool's calls or results.
- **Text search** across all blocks, including system and tools, with highlighting and next/previous.

### 8.4 Errors easy to spot
- Error count badge in the top bar. Clicking it jumps to the next error.
- Cards with an HTTP error, an upstream failure, or any `is_error` tool result get a red left edge and an `ERROR` badge in the header.
- Filter "Errors only".

### 8.5 Drill-down
- Every block, section and exchange can be opened: collapsed → expanded → JSON tree → raw.
- "Copy JSON" on every block, section and exchange.

### 8.6 Keyboard
- `Home` jump to newest · `Space` pause/resume · `/` focus search · `c` collapse all · `e` expand all · `n` next error.

### 8.7 Performance
- Stays smooth with ≥ 500 exchanges (collapsed sections render lazily on expand; the list is windowed).
- A new exchange card appears within 200 ms of the proxy receiving the request.

## 9. Security and privacy
- The dashboard binds to `127.0.0.1` only.
- `authorization` and `x-api-key` are redacted before anything is stored or sent to the browser.
- Traffic may contain file contents and secrets, so nothing is written to disk unless logging is explicitly enabled with `--log` (see D5).

## 10. Acceptance criteria
1. Starting the proxy + dashboard and running `ANTHROPIC_BASE_URL=http://localhost:8041 claude` shows a new exchange card **at the top** within 1 s of each request.
2. Every request the proxy forwards, including `count_tokens` and utility calls, appears as a card. The card count equals the request count.
3. Each `/v1/messages` card shows SYSTEM (full text), TOOLS (every definition), CONVERSATION SO FAR, and NEW MESSAGE as four visibly separate sections, followed by REPLY.
4. When the agent runs `ls`: the reply to request N shows `TOOL ▸ Bash ls` with a waiting placeholder beside it.
   When request N+1 arrives, the result appears **beside** the call, and both link to each other's exchange.
5. The reply section shows the model's text and every tool call it requested, plus `stop_reason`.
6. The stats panel shows tokens per request, estimated cost, time, and per-tool usage, and clicking any of them leads to the underlying messages.
7. Every message can be expanded down to pretty-printed JSON and the raw SSE events.
8. The Diff tab shows exactly the messages added since the previous request, and flags any change to system, tools or model.
9. An HTTP error or `is_error` tool result is visible from the top bar badge and jumps to the card in one click.
10. No full token value ever appears in the UI or the browser's network traffic.
11. The proxy still forwards traffic byte-for-byte, and the agent behaves the same with the dashboard open or closed.
12. The UI is light-themed throughout and every text/background pair passes WCAG AA.

## 11. Technical decisions

| # | Decision | Options considered | Chosen |
|---|---|---|---|
| D1 | Where the dashboard lives | (a) Add it to `proxy.py`, which also serves the web page; (b) the proxy writes a log file and a separate dashboard program reads it; (c) a second proxy in front of the first | **(a)**: one process, one command. Serve the page on a second port (default 8042) so the agent's traffic and the page don't mix |
| D2 | Pushing updates to the browser | SSE / WebSocket / polling | **SSE**: Python's standard library can do it, it only needs one direction, and it reconnects automatically |
| D3 | Frontend tech | One HTML file with plain JavaScript / React or Svelte with a build step | **One file, no build**, matching the lab's "standard library only, nothing to install" spirit |
| D4 | Telling conversations apart (the API sends no conversation ID) | Hash of system prompt + first message / message-history prefix matching / flat list | **Hash plus prefix matching**: if a new request's history starts with a known conversation's history, it belongs to that conversation. "New message" and "Diff" are computed against the previous request in that conversation |
| D5 | Saving traffic | Memory only / optional `--log traffic.jsonl` with replay on load | **Memory only by default, plus an optional `--log` flag** |
| D6 | Where parsing happens | In the Python proxy (send clean events) / in the browser (send raw bytes) | **In the proxy**: the browser gets structured events, and redaction happens before anything leaves Python. The raw body and SSE events are also kept for the Raw tab |
| D7 | Codex / OpenAI support | v1 Anthropic only / both | **Anthropic only in v1**, with the parser designed so a Codex parser can be added later |
| D8 | Telling typed prompts from injected text | Show as YOU / separate `CONTEXT` style / hide | **Separate `CONTEXT` style, collapsed** |
| D9 | Sending large repeated parts (system, tools, history) to the browser on every request | Send everything each time / send each unique part once (by content hash) and reference it | **Send each unique part once**, so the browser stays fast while still showing every request in full |
