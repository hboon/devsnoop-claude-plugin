# DevSnoop plugin for Claude Code

This plugin teaches Claude Code how to use an existing DevSnoop installation to inspect and interact with pages in your current or selected Chrome profile. It contains one skill and no executable code.

DevSnoop is a commercial product made by the author of this plugin. DevSnoop is not made, sponsored, or endorsed by Anthropic, and this repository does not make DevSnoop a first-party Claude integration.

## What the plugin contains

- `.claude-plugin/plugin.json`: Claude Code plugin metadata.
- `skills/devsnoop/SKILL.md`: instructions for calling the local DevSnoop bridge.

The plugin has no hooks, MCP server, LSP server, executable, installer, background process, remote code, bundled binary, dependency, or automatic setup.

## Separate DevSnoop setup

Install the DevSnoop Chrome extension and native host separately using the [DevSnoop installation guide](https://devsnoop.com/install?utm_source=claude_plugin_directory&utm_medium=plugin&utm_campaign=devsnoop_claude_launch&utm_content=setup_skill).

DevSnoop supports macOS and Linux. Windows is not supported. The product includes one 24-hour preview with all commands enabled. A $29 one-time license keeps DevSnoop working after the preview and includes one year of updates.

The native host exposes a local HTTP bridge on `127.0.0.1:9400`. DevSnoop works with the user's current or selected Chrome profile; it does not create an isolated browser. Chrome asks the user to approve page access for an exact origin, such as `https://shop.example.com/*`. The extension declares `<all_urls>` as an optional host permission only so Chrome can request these per-origin grants at runtime.

Read the [DevSnoop privacy policy](https://devsnoop.com/privacy) for the product's data handling, license verification, sanitized diagnostics, analytics, and local file retention.

DevSnoop returns command data to the local native host. When Claude Code reads a command response or opens a DevSnoop screenshot, that selected data is processed as part of the user's Claude Code interaction under Anthropic's terms and privacy policy.

## Work and test Chrome profiles

Use DevSnoop to check the account in the Chrome profile you intend. Install the extension in each profile, then give each one a distinct **Agent profile label** in its popup, such as Work or Test account (generic examples). Keep the profiles connected. Each profile needs an active preview or license and its own site-origin grants.

Ask Claude Code:

> Use DevSnoop to list connected profiles. Use the profile labeled Test account to inspect the requested test page. Include its connectionID on every command, check ping, then list its tabs and use the matching tabID. Ask me if the profile or tab is unclear. Do not inspect unrelated tabs or storage.

`list_connections` returns labels, local `profileID` values and current `connectionID` values. `select_connection` changes the broker's shared default. For account-specific work, a top-level `connectionID` on every request avoids relying on that default. Connection IDs can change on reconnect; if a target disappears, list connections again and confirm it before continuing.

See the [multi-profile workflow](https://devsnoop.com/coding-agent-multiple-chrome-profiles) for HTTP examples, default fallback behavior and permissions. DevSnoop uses existing profiles; it does not create an isolated browser. Labels help choose a profile and are not access restrictions.

## Example prompts

These prompts assume DevSnoop is already installed and connected.

> Use DevSnoop to diagnose why the checkout button click fails on my current Chrome tab. Click it once, then report the selector, visible page change, console errors, and failed network requests.

> Use DevSnoop on the current Chrome tab. Submit the form with the values already entered, then inspect console exceptions and failed network requests. Do not inspect unrelated tabs or storage.

> Use DevSnoop to check the current page's legibility. Run the legibility audit, capture screenshots of the page and any flagged element, inspect the images, and report only problems you can verify.

## Local development and review

Validate the repository with the current Claude Code CLI:

```bash
claude plugin validate . --strict
```

Load it for a local test without publishing or installing it:

```bash
claude --plugin-dir .
```

The namespaced skill is `/devsnoop:devsnoop`.

For an Anthropic community-marketplace review, DevSnoop does not require a user account. A reviewer can install the separate product, start the 24-hour full-command preview, and use any local or public test page with non-sensitive sample data. The plugin repository must be public before submission, and the submitted commit must contain the same files that were reviewed locally.

## Support and security

Report product, plugin, privacy, or security issues to [hboon@devsnoop.com](mailto:hboon@devsnoop.com). Include the plugin version and a minimal reproduction. Do not email secrets, license keys, page content, screenshots, cookies, storage values, or network payloads.

The skill-only wrapper in this repository is available under the MIT License. The DevSnoop Chrome extension, native host, service, and commercial license are separate and are not licensed under this repository's MIT License.
