# Privacy Policy — Swallow Pricing Engine Plugin

**Effective date:** 25 March 2025
**Plugin name:** swallow-pricing-engine
**Provider:** Llow Group Ltd
**Contact:** dev@llow.io

## What this plugin does

This plugin connects Claude Code to the Swallow Pricing Engine API via the Model Context Protocol (MCP). It sends insurance pricing model configurations to the API for validation, testing, and schema retrieval.

## Data sent to the API

When you use the MCP tools provided by this plugin, the following data is sent to `https://api.llow.io/ai/mcp`:

| Tool | Data sent |
|------|-----------|
| `validate_swallow_project` | The project configuration JSON you provide |
| `test_swallow_project` | The project configuration JSON including test cases |
| `schema_swallow_project` | No data sent (schema fetch only) |

**No personal data is collected.** The plugin sends only the pricing model JSON that you explicitly pass to the MCP tools. It does not send your name, email, API keys, file paths, or any data from your local filesystem beyond what you directly include in a tool call.

## Data stored by the API

- Project configurations sent to `validate_swallow_project` and `test_swallow_project` are **processed in memory and not stored**. No data is written to a database or persisted on the server.
- Standard HTTP access logs (IP address, timestamp, request size) may be retained for up to 30 days for operational monitoring.

## Data stored locally

- Plugin settings (non-sensitive) are stored in your Claude Code `settings.json`.
- No other local data is created or stored by this plugin.

## Third-party services

- **Swallow API** (`api.llow.io`) — hosted on AWS (eu-west-2, London). Subject to Llow Group's infrastructure security practices.
- **No analytics, tracking, or telemetry** is included in this plugin.
- **No data is shared with third parties** beyond the API endpoint above.

## Your rights

- You control what data is sent — the plugin only transmits data you explicitly pass to MCP tools.
- You can stop all data transmission by disabling or uninstalling the plugin.
- To request deletion of access logs, contact contact@llow.io.

## Changes to this policy

Updates will be published to this file in the plugin repository. The effective date at the top will be updated accordingly.

## Contact

For privacy questions: contact@llow.io
