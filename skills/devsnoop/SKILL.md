---
name: devsnoop
description: Use an existing DevSnoop installation to inspect, interact with, or debug pages in the user's current Chrome profile. Invoke only when the user explicitly asks to use DevSnoop or asks Claude to control their live Chrome browser with DevSnoop.
---

# DevSnoop browser access

DevSnoop lets Claude Code inspect and interact with pages in the user's current or selected Chrome profile. The DevSnoop extension, native host, and service are separate commercial products and are not made, sponsored, or endorsed by Anthropic.

The Claude plugin contains instructions only. It does not install or bundle the DevSnoop Chrome extension or native host. If the local bridge is unavailable, direct the user to the [DevSnoop installation guide](https://devsnoop.com/install?utm_source=claude_plugin_directory&utm_medium=plugin&utm_campaign=devsnoop_claude_launch&utm_content=setup_skill). Do not install anything automatically.

DevSnoop supports macOS and Linux. It does not support Windows. Users can run every command during one 24-hour preview. A $29 one-time license keeps DevSnoop working after the preview and includes one year of updates.

## Safety and privacy boundaries

- Send commands only after the user requests a DevSnoop browser workflow.
- The native host listens on `127.0.0.1`, so the bridge is local to the user's machine. The product separately contacts DevSnoop for license verification and sanitized connection diagnostics, and PostHog for limited product analytics as described in the [DevSnoop privacy policy](https://devsnoop.com/privacy).
- DevSnoop works in the user's current or explicitly selected Chrome profile. It is not an isolated browser. `list_tabs` can return the open tabs in that profile.
- Chrome grants page access one origin at a time. For example, access to `https://shop.example.com` is granted as `https://shop.example.com/*`. Never describe a grant as broader than the origin Chrome shows.
- The extension declares `optional_host_permissions` as `<all_urls>` so Chrome can offer an exact runtime origin grant. It does not receive every-site access at installation.
- `get_logs`, `get_network`, and screenshots use Chrome's `debugger` permission. Tell the user when Chrome asks for a permission or site-access decision. Do not bypass or weaken it.
- Do not inspect unrelated tabs, origins, storage, console output, network traffic, or screenshots. Use only the data needed for the user's request.
- DevSnoop returns command responses to the local native host. Claude Code can process the response text and any screenshot opened with its image-reading tool as part of the user's Claude Code interaction. Keep requests to the data needed for the task, and do not forward it to any additional service unless the user separately asks.
- Keep sensitive storage values redacted. `get_storage` defaults to `redactValues: true`; do not disable redaction unless the user explicitly asks and the task requires it.

## Connect and check entitlement

Send JSON requests to `http://127.0.0.1:9400/`. The port can be changed with the native host's `DEVSNOOP_PORT` environment variable.

```bash
curl -sS -X POST http://127.0.0.1:9400/ \
  -H 'Content-Type: application/json' \
  -d '{"command":"ping","params":{}}'
```

Check entitlement with `ping` in the intended profile before page commands. When multiple profiles are connected, identify the profile first as described below.

- If `license.activated` is `true`, continue.
- If `license.activated` is `false` and `preview.active` is `true`, continue until `preview.expiresAt`.
- If `preview.reason` is `"not_started"`, ask the user to open the DevSnoop extension popup, click **Start preview**, and approve the exact site origin if Chrome asks. Run `ping` again after confirmation.
- If `preview.reason` is `"expired"`, explain that the 24-hour preview has expired and a license must be bought or activated before continuing.
- If the bridge does not respond, explain that the separate Chrome extension and native host must be installed. Link to the installation guide above. Do not download or run the installer yourself.

## Select a Chrome profile

For one connected profile, no selection is needed unless the user requested a particular account. For multiple profiles, call `list_connections` before the entitlement `ping`. Match the user's intended profile by its optional label or known `profileID`; ask if the match is unclear. Labels can be empty or duplicated. Never copy unrelated profile labels, tab lists or page data into a report.

```json
{ "command": "list_connections", "params": {} }
```

The response's `result` includes `selectedConnectionID` and `connections`. Each connection includes `connectionID`, `profileID`, `label` and `selected`, plus connection metadata. Profile IDs are stored in local extension storage; connection IDs can change on reconnect. Use the current returned connection ID, not the profile ID, to send commands.

The IDs and labels in these examples are generic. Replace `native-456-example` with the intended connection's current ID, and `123` with a tab `id` returned by that connection's `list_tabs`.

For account-specific workflows, include a top-level `connectionID` on every request, including entitlement and tab discovery:

```json
{ "command": "ping", "params": {}, "connectionID": "native-456-example" }
```

Apply the license and preview checks above to that profile's `ping` response before continuing.

```json
{ "command": "list_tabs", "params": {}, "connectionID": "native-456-example" }
```

```json
{
  "command": "page_summary",
  "params": {},
  "connectionID": "native-456-example",
  "tabID": 123
}
```

Confirm that the summary matches the intended account and page before an interaction. Keep both IDs on subsequent page commands. Omitting `tabID` uses that profile's active tab.

To change the default for commands that omit `connectionID`:

```json
{
  "command": "select_connection",
  "params": { "connectionID": "native-456-example" }
}
```

This default is shared by callers using the broker and held in memory. Broker restarts reset it. Per-request targeting overrides it without changing it. If the default connection disappears, untargeted commands can use another available connection. A stale per-request connection ID returns an error instead of falling back. List connections again and confirm the intended profile after a disconnect; do not blindly retry an action whose outcome is unknown.

Profile selection does not grant page access. If an origin permission is needed, ask the user to grant it in that profile's popup. Labels are stored locally and shown to local callers; they do not restrict access. Chrome can show a debugging banner for debugger-backed commands. The bridge's agent HTTP API has no token requirement: use trusted local callers and do not expose its loopback port to a network.

See the [multi-profile workflow](https://devsnoop.com/coding-agent-multiple-chrome-profiles) for setup and HTTP examples. Preserve the safety and privacy boundaries above.

## Request format

```bash
curl -sS -X POST http://127.0.0.1:9400/ \
  -H 'Content-Type: application/json' \
  -d '{"command":"<command>","params":{},"tabID":123}'
```

Put `tabID` at the top level and omit it from `params`. Omit `tabID` entirely only when the active tab is intentionally the target. Once a tab ID is known, include it in every page command.

## Commands

### Tab management

**`list_tabs`** returns the open tabs in the current or selected Chrome profile.

```json
{ "command": "list_tabs", "params": {} }
```

Returns: `[{id, url, title, active}]`

**`open_tab`** opens a tab, optionally with a URL, and waits for the page to load.

```json
{ "command": "open_tab", "params": { "url": "https://example.com" } }
```

Returns: `{tabID, url, title}`

**`navigate`** changes the URL of a tab and waits for the page to load.

```json
{
  "command": "navigate",
  "params": { "url": "https://example.com/page" },
  "tabID": 123
}
```

Returns: `{tabID, url, title}`

**`close_tab`** closes the named tab. Omit `tabID` only when the active tab is intentionally the target.

```json
{ "command": "close_tab", "params": {}, "tabID": 123 }
```

Returns: `{closed, tabID, url, title}`

### Page inspection

**`page_summary`** returns a semantic DOM tree with interactive elements and ready-to-use selectors. Start page inspection here.

```json
{ "command": "page_summary", "params": { "depth": 3 }, "tabID": 123 }
```

Params: `depth` (optional, default 4).
Returns: nested regions with `{tag, role, id, classes, textPreview, children, interactiveElements}`. Interactive elements include `{tag, type, label, selector, value}`.

**`inspect_element`** returns one element's HTML subtree and relevant computed styles.

```json
{
  "command": "inspect_element",
  "params": { "selector": "#my-form", "depth": 2, "includeStyles": true },
  "tabID": 123
}
```

Params: `selector` or `text` (one required), `depth` (optional, default 3), and `includeStyles` (optional, default true).
Returns: `{element, subtree, styles}`, with styles grouped as `{layout, sizing, typography, visual}`.

**`find`** queries elements by CSS selector or visible text.

```json
{ "command": "find", "params": { "text": "Submit order" }, "tabID": 123 }
```

Params: `selector` or `text` (one required) and `limit` (optional, default 20). Text search prioritizes interactive elements and exact matches.
Returns: `{elements: [{tag, textContent, selector, boundingRect, classes, attributes}], totalCount}`.

**`tech_stack`** detects frameworks, libraries, CMSs, and analytics.

```json
{ "command": "tech_stack", "params": {}, "tabID": 123 }
```

Returns: `{detections: [{name, category, confidence, version}], count}`.

**`get_storage`** reads local or session storage from the requested page.

```json
{
  "command": "get_storage",
  "params": { "type": "local", "keys": ["user"] },
  "tabID": 123
}
```

Params: `type` (optional: `"local"`, `"session"`, or `"all"`; default `"all"`), `keys` (optional), `redactValues` (optional, default true), and `redactKeys` (optional extra key substrings to redact).
Returns: `{localStorage, sessionStorage}` with sensitive-looking values redacted by default. Do not set `redactValues` to false unless the user explicitly requests it and the task requires the values.

**`accessibility_audit`** runs built-in accessibility checks.

```json
{ "command": "accessibility_audit", "params": {}, "tabID": 123 }
```

Returns: `{issues, score, totalIssues}`. Checks include missing alt text, unlabeled inputs, skipped headings, duplicate IDs, and missing landmarks.

**`legibility_audit`** runs a heuristic visual QA pass for weak contrast, tiny or clipped text, cramped lines, low-contrast icons, and text over images or gradients.

```json
{
  "command": "legibility_audit",
  "params": { "viewportOnly": true, "maxIssues": 40 },
  "tabID": 123
}
```

Params: `maxIssues` (optional, default 40) and `viewportOnly` (optional, default true).
Returns: `{issues, score, summary, checkedTextNodes}`. Issues include visible text or an icon label, location, a verified selector when available, style details, and a suggested fix. This is a legibility smoke test, not accessibility certification.

**`screenshot`** saves a full-tab or selected-element image to a local file. It can capture a tab by ID even when that tab is not focused.

```json
{ "command": "screenshot", "params": { "format": "png" }, "tabID": 123 }
```

Params: `format` (optional: `"png"` or `"jpeg"`), `quality` (optional, default 90), and `selector` (optional).
Returns: `{dataURL, elementBounds, width, height}`. The `dataURL` field contains a local file path. Open and inspect that file with Claude's image-reading tool before making a visual claim.

For user-visible web changes, capture and inspect the affected route, representative viewport sizes, and states that the change can affect. Pair screenshots with interaction, logs, network, accessibility, or legibility checks when relevant. Keep captures temporary, remove verification-only overrides, and close tabs opened only for the task. Nonvisual changes do not require screenshots.

### Page interactions

**`click`** clicks by selector, visible text, or coordinates.

```json
{ "command": "click", "params": { "text": "Submit order" }, "tabID": 123 }
```

Params: `selector`, `text`, or `x` plus `y` (one method required). Text search prioritizes interactive elements.
Returns: `{clicked, element, selector}`. A successful command means the click was dispatched; verify the intended page change separately.

**`fill`** sets the value of an input, textarea, select, or contenteditable element and fires input and change events.

```json
{
  "command": "fill",
  "params": { "selector": "#email", "value": "test@example.com" },
  "tabID": 123
}
```

Params: `selector` or `text` (one required), plus `value`.
Returns: `{filled, element, previousValue}`.

**`scroll`** scrolls the page or brings an element into view.

```json
{
  "command": "scroll",
  "params": { "direction": "down", "amount": 300 },
  "tabID": 123
}
```

Params: `direction` plus optional `amount` (default 300), or `selector`.
Returns: `{scrolled, position: {x, y}}`.

**`hover`** triggers hover events on an element.

```json
{ "command": "hover", "params": { "text": "Menu" }, "tabID": 123 }
```

Params: `selector` or `text` (one required).
Returns: `{hovered, element}`.

### Debugging

**`get_logs`** returns buffered console messages and JavaScript exceptions. It attaches Chrome's debugger on first use.

```json
{
  "command": "get_logs",
  "params": { "level": "error", "limit": 10 },
  "tabID": 123
}
```

Params: `level`, `since` (Unix milliseconds), and `limit`, all optional.
Returns: `{logs: [{level, text, url, line, timestamp}], exceptions: [{text, description, stackTrace, timestamp}]}`. DevSnoop buffers up to 200 entries, so start capture before the event when complete logs matter.

**`get_network`** returns network requests with filters and optional redacted request or response bodies. It attaches Chrome's debugger on first use.

```json
{
  "command": "get_network",
  "params": { "urlPattern": "api", "errorsOnly": true },
  "tabID": 123
}
```

Params: `status`, `type`, `urlPattern`, `errorsOnly`, and `limit`, all optional. Body capture is off by default. `includeRequestBody` and `includeResponseBody` opt in to bodies; `bodyMaxBytes` defaults to and is capped at 20,000; `redactBody` defaults to true; and `redactKeys` adds key substrings to redact.
Returns: `{requests: [{method, url, status, type, duration, size, error, requestBody, responseBody}]}`. Do not disable body redaction unless the user explicitly requests it and the task requires the unredacted data.

### Change detection

**`diff`** captures a DOM baseline on the first call and reports later changes on the second.

```json
{ "command": "diff", "params": {}, "tabID": 123 }
```

First call returns: `{baseline: true, message, timestamp, url}`.
Second call returns: `{changes: [{type, path, details}], summary: {added, removed, changed}}`.

### Health check

**`ping`** verifies the connection, extension version, license, and preview state.

```json
{ "command": "ping", "params": {} }
```

Returns: `{pong, timestamp, version, license, preview}`. When the preview is inactive, `preview.reason` is `"not_started"` or `"expired"`.

Use `page_summary` or `find` to obtain a selector before interacting. Prefer visible text when the button or link label is known. After a click or form submission, inspect page changes plus relevant console and network failures.

## Permission handling

If a response says that permission is needed for an origin, stop page commands for that origin. Tell the user to open the DevSnoop popup and grant the exact origin Chrome displays. Resume only after the user confirms and a retried command succeeds.

Do not ask for access to every site. A grant for one origin does not authorize inspecting another origin, even in the same profile.

## Workflow examples

### Diagnose a failed checkout click

1. Run `ping`, then `list_tabs` and identify the requested checkout tab.
2. Run `page_summary` with its `tabID` and find the checkout button.
3. Call `click` using the returned selector or the exact visible label.
4. Run `get_logs` with `level: "error"`, `get_network` with `errorsOnly: true`, and `diff` or another `page_summary`.
5. Report the clicked selector, visible page change, console exception, and failed request. Do not claim the click succeeded only because the click command returned.

### Inspect failures after form submission

1. Run `get_logs` and `get_network` before submission if the user needs the complete event.
2. Fill only the fields the user requested and submit the form.
3. Run `get_logs` for errors and `get_network` with `errorsOnly: true` on the same `tabID`.
4. Correlate the form response, console exception, and visible state without exposing unrelated request data.

### Verify legibility with screenshots

1. Run `legibility_audit` on the requested tab.
2. Capture a full-tab or element screenshot.
3. Open and inspect the saved image with Claude's image-reading tool.
4. Report concrete text, icon, contrast, clipping, or spacing problems. Keep heuristic findings separate from what is visible in the screenshot.

## Troubleshooting

- No response from `127.0.0.1:9400`: the native host is not installed or running, the extension is not connected, or a custom port is configured.
- Version warning from `ping`: the extension and native host versions differ. Follow the matching-host instruction returned by DevSnoop only after showing it to the user.
- Origin permission error: the user must grant that exact origin in the extension popup.
- Empty logs: capture may have started after the event. Start `get_logs` before repeating the requested interaction when the user permits a repeat.
- Multiple profiles: confirm the selected connection rather than assuming the frontmost Chrome window is the target.

## Command tips

- Include `tabID` after identifying the target tab so a focus change cannot redirect a later command.
- Use `text` when the requested button or link label is known. Use selectors returned by `page_summary` or `find` for the other commands.
- Reuse the selector returned by `click` when a later check targets the same element.
- Take a `diff` baseline before the action or hot reload that needs to be compared.
