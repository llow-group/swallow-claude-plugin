# Swallow Pricing Engine Plugin

A Claude Code plugin that connects to the [Swallow](https://swallow.app) insurance pricing engine via MCP. Build, validate, analyse, and test insurance pricing models directly from Claude.

There are two ways to use this project — as a **full plugin** (Claude Code only) or as a standalone **MCP server** (Claude Chat, Claude Code, or any MCP client). The plugin includes the MCP server, so you don't need to set up both.

| | Plugin | MCP Server |
|---|---|---|
| **Platform** | Claude Code | Claude Chat, Claude Code, any MCP client |
| **MCP Tools** | Yes | Yes |
| **MCP Prompts** | Yes | Yes |
| **MCP Resource** | Yes | Yes |
| **Skills** | Yes | No |
| **Agents** | Yes | No |
| **Documentation** | Yes (bundled) | Yes (via MCP) |

---

## Plugin (Claude Code)

The full plugin gives you everything: MCP tools, prompts, skills, agents, and bundled documentation.

### Installation

#### From the marketplace

Add the marketplace and install the plugin:

```
/plugin marketplace add llow-group/swallow-claude-plugin
/plugin install swallow-pricing-engine@swallow-pricing-engine
/reload-plugins
```

Or add it manually to your Claude Code settings (`~/.claude/settings.json`):

```json
{
  "extraKnownMarketplaces": {
    "swallow": {
      "source": "github",
      "repo": "llow-group/swallow-claude-plugin"
    }
  }
}
```

Then install:

```
/plugin install swallow-pricing-engine@swallow
```

You can choose your installation scope:
- **User** — available across all projects
- **Project** — shared with collaborators (stored in `.claude/settings.json`)
- **Local** — just for you in this repo

#### From the git repo (local development)

Clone the repo and load it directly:

```bash
git clone https://github.com/llow-group/swallow-claude-plugin.git
claude --plugin-dir ./swallow-claude-plugin
```

This is useful for development, testing, or customising the plugin before sharing.

### Skills

| Skill | Description |
|-------|-------------|
| `/swallow-pricing-engine:build-pricing-model` | Guided workflow for building a Swallow config from a product description |
| `/swallow-pricing-engine:excel-to-swallow` | Convert an Excel workbook to Swallow JSON by tracing formulas |
| `/swallow-pricing-engine:validate-project` | Run deep schema validation on a project file |
| `/swallow-pricing-engine:test-project` | Run and debug embedded tests |
| `/swallow-pricing-engine:swallow-docs` | Load documentation, JSON schema, and agent rules into context |

### Agents

| Agent | Role | When to use |
|-------|------|-------------|
| `swallow-actuary` | Builds technically correct pricing models | Creating new models, fixing validation errors, adding steps/factors |
| `swallow-analyst` | Reviews models commercially | Understanding pricing behaviour, competitive positioning, recommending improvements |

The actuary builds the model. The analyst tells you if the model makes commercial sense.

Both agents call `docs_swallow_project` to load the latest rules and constraints before working.

### Usage examples

#### Build a new pricing model

```
@swallow-actuary Build a motor insurance model with age, vehicle value,
postcode, NCD, and conviction count as rating factors
```

Or via skill:

```
/swallow-pricing-engine:build-pricing-model

I need a pet insurance pricing model. Factors: animal species (dog/cat),
breed, age, neutered status, and owner postcode. Exclude dogs over 12 and
cats over 18. Base premium is 500, with breed and age loading factors.
```

#### Analyse an existing model

```
@swallow-analyst Review ./motor-insurance.json for a UK direct-to-consumer
audience aged 25-45. Tell me where we're competitive and where we're exposed.
```

The analyst will:
1. Generate ~2,500 test cases sweeping across all rating factors
2. Run them through the engine via MCP
3. Map the pricing surface (percentiles, factor sensitivity, cliff edges)
4. Assess competitiveness for your target market
5. Recommend retail and technical pricing improvements

#### Convert an Excel rater

```
/swallow-pricing-engine:excel-to-swallow

Convert ./motor-rater.xlsx to a Swallow model. The result cell is premium!H45.
```

Or let it find the result automatically via a `:::result` comment in the spreadsheet:

```
/swallow-pricing-engine:excel-to-swallow

Convert ./motor-rater.xlsx — the result cell is marked with :::result
```

Claude reads the workbook, traces all formula dependencies backwards from the result cell, classifies each cell (input, constant, lookup table, calculation, exclusion), translates formulas to Swallow expressions, builds the JSON, then validates and tests via MCP.

#### Reference the engine docs

```
/swallow-pricing-engine:swallow-docs
```

Loads the Swallow Pricing Engine documentation, JSON schema, and agent rules into context — step types, expression syntax, configuration constraints, and more.

#### Validate an existing project

```
/swallow-pricing-engine:validate-project

Check the project in ./my-project.json
```

#### Run tests

```
/swallow-pricing-engine:test-project

Run the tests in ./my-project.json and tell me what's failing
```

---

## MCP Server

The MCP server gives you the 4 tools, 4 workflow prompts, and a guide resource. It works with Claude Chat, Claude Code, or any MCP client — no plugin installation required.

### Setup

#### Claude Chat (via Connectors)

1. Open [claude.ai](https://claude.ai) and go to **Settings**
2. Select **Connectors** from the sidebar
3. Click **Add Connector**
4. Enter the MCP server URL: `https://api.llow.io/ai/mcp`
5. Give it a name (e.g. "Swallow") and save

Once connected, the MCP tools and workflow prompts will be available in your Claude Chat conversations.

#### Claude Code (via settings.json)

If you want the MCP tools without the full plugin, add the server to your Claude Code settings file. Choose the scope that suits you:

- **User** (`~/.claude/settings.json`) — available across all projects
- **Project** (`.claude/settings.json` in your repo) — shared with collaborators
- **Local** (`.claude/settings.local.json` in your repo) — just for you in this repo

```json
{
  "mcpServers": {
    "swallow": {
      "type": "http",
      "url": "https://api.llow.io/ai/mcp"
    }
  }
}
```

After saving, restart Claude Code or run `/mcp` to verify the server is connected. You should see the 4 Swallow tools available.

#### Other MCP clients

Add the MCP server as an integration pointing to:

```
https://api.llow.io/ai/mcp
```

### Tools

| Tool | Description |
|------|-------------|
| `test_swallow_project` | Run embedded tests through the pricing engine and return actual prices |
| `validate_swallow_project` | Deep schema validation — returns errors with JSON path, keyword, and message |
| `schema_swallow_project` | Fetch the full JSON schema for project config files |
| `docs_swallow_project` | Fetch documentation by topic: `"rules"` for agent rules and constraints, `"readme"` for full engine reference |

### Prompts

Available in Claude Chat and any MCP client that supports prompts:

| Prompt | Description |
|--------|-------------|
| `build-pricing-model` | Guided workflow for building a Swallow config from a product description |
| `analyse-pricing-model` | Stress-test a model with ~2,500 cases and produce a pricing surface report |
| `test-pricing-model` | Run and debug embedded tests with a common issues lookup table |
| `convert-excel-rater` | Convert an Excel workbook to Swallow JSON by tracing formulas |

### Resource

| URI | Description |
|-----|-------------|
| `swallow://guide` | Comprehensive reference guide for the Swallow Pricing Engine |

---

## Plugin Structure

```
swallow-claude-plugin/
├── .claude-plugin/
│   ├── plugin.json            # Plugin manifest
│   └── marketplace.json       # Marketplace listing
├── .mcp.json                  # MCP server connection
├── docs/
│   ├── swallow_readme.md      # Full engine reference
│   ├── swallow_schema.json    # Project JSON schema
│   └── swallow_Agent_rules.md # Agent rules and constraints
├── prompts/
│   ├── build-pricing-model.md    # MCP prompt: build from description
│   ├── analyse-pricing-model.md  # MCP prompt: stress-test and analyse
│   ├── test-pricing-model.md     # MCP prompt: run and debug tests
│   └── convert-excel-rater.md    # MCP prompt: Excel to Swallow
├── skills/
│   ├── build-pricing-model/   # Build from description
│   ├── excel-to-swallow/      # Convert Excel rater
│   ├── validate-project/      # Schema validation
│   ├── test-project/          # Run and analyse tests
│   └── swallow-docs/          # Load documentation and schema
├── agents/
│   ├── swallow-actuary.md     # Technical pricing builder
│   └── swallow-analyst.md     # Commercial pricing analyst
├── LICENSE
├── PRIVACY.md
└── README.md
```

## API Endpoint

The plugin connects to `https://api.llow.io/ai/mcp`. For self-hosted deployments, update the URL in `.mcp.json`.

## License

MIT — Llow Group
