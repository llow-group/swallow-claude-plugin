# Swallow Pricing Engine Plugin

A Claude Code plugin that connects to the [Swallow](https://swallow.app) insurance pricing engine via MCP. Build, validate, analyse, and test insurance pricing models directly from Claude.

## What you get

- **MCP Server** — 4 tools: test project, validate project, fetch schema, fetch documentation
- **MCP Prompts** — 4 workflow prompts for building, testing, analysing, and converting pricing models (available in claude.ai via MCP integration)
- **MCP Resource** — `swallow://guide` comprehensive reference guide
- **Skills** — `/swallow-pricing-engine:build-pricing-model`, `/swallow-pricing-engine:excel-to-swallow`, `/swallow-pricing-engine:validate-project`, `/swallow-pricing-engine:test-project`, `/swallow-pricing-engine:swallow-docs`
- **Agents** — `swallow-actuary` (builds models) and `swallow-analyst` (reviews and stress-tests them)
- **Documentation** — Engine reference, JSON schema, and agent rules in `docs/`

## Installation

### From the marketplace

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

### From the git repo (local development)

Clone the repo and load it directly:

```bash
git clone https://github.com/llow-group/swallow-claude-plugin.git
claude --plugin-dir ./swallow-claude-plugin
```

This is useful for development, testing, or customising the plugin before sharing.

### MCP server only

If you only want the MCP tools and prompts without the full plugin, you can add the MCP server directly.

#### Claude Code (via settings.json)

Add the following to your Claude Code settings file. Choose the scope that suits you:

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

#### claude.ai / other MCP clients

Add the MCP server as an integration pointing to:

```
https://api.llow.io/ai/mcp
```

This gives you the 4 MCP tools, 4 workflow prompts, and the guide resource — but not the local skills or agents.

## Usage

### Build a new pricing model

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

### Analyse an existing model

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

### Convert an Excel rater

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

### Reference the engine docs

```
/swallow-pricing-engine:swallow-docs
```

Loads the Swallow Pricing Engine documentation, JSON schema, and agent rules into context — step types, expression syntax, configuration constraints, and more.

### Validate an existing project

```
/swallow-pricing-engine:validate-project

Check the project in ./my-project.json
```

### Run tests

```
/swallow-pricing-engine:test-project

Run the tests in ./my-project.json and tell me what's failing
```

## Agents

| Agent | Role | When to use |
|-------|------|-------------|
| `swallow-actuary` | Builds technically correct pricing models | Creating new models, fixing validation errors, adding steps/factors |
| `swallow-analyst` | Reviews models commercially | Understanding pricing behaviour, competitive positioning, recommending improvements |

The actuary builds the model. The analyst tells you if the model makes commercial sense.

Both agents call `docs_swallow_project` to load the latest rules and constraints before working.

## MCP Tools

| Tool | Description |
|------|-------------|
| `test_swallow_project` | Run embedded tests through the pricing engine and return actual prices |
| `validate_swallow_project` | Deep schema validation — returns errors with JSON path, keyword, and message |
| `schema_swallow_project` | Fetch the full JSON schema for project config files |
| `docs_swallow_project` | Fetch documentation by topic: `"rules"` for agent rules and constraints, `"readme"` for full engine reference |

## MCP Prompts

Available in claude.ai and any MCP client that supports prompts:

| Prompt | Description |
|--------|-------------|
| `build-pricing-model` | Guided workflow for building a Swallow config from a product description |
| `analyse-pricing-model` | Stress-test a model with ~2,500 cases and produce a pricing surface report |
| `test-pricing-model` | Run and debug embedded tests with a common issues lookup table |
| `convert-excel-rater` | Convert an Excel workbook to Swallow JSON by tracing formulas |

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
