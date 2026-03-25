# Swallow Pricing Engine Plugin

A Claude Code plugin that connects to the [Swallow](https://swallow.app) insurance pricing engine via MCP. Build, validate, analyse, and test insurance pricing models directly from Claude.

## What you get

- **MCP Server** — 3 tools: test project, validate project, fetch schema
- **Commands** — `/swallow-pricing-engine:docs` for full engine reference
- **Skills** — `/swallow-pricing-engine:build-pricing-model`, `/swallow-pricing-engine:excel-to-swallow`, `/swallow-pricing-engine:validate-project`, `/swallow-pricing-engine:test-project`
- **Agents** — `swallow-actuary` (builds models) and `swallow-analyst` (reviews and stress-tests them)

## Installation

Add the marketplace to your Claude Code settings (`~/.claude/settings.json`):

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

```bash
claude plugin install swallow-pricing-engine@swallow
```

### Test locally

```bash
git clone https://github.com/llow-group/swallow-claude-plugin.git
claude --plugin-dir ./swallow-claude-plugin
```

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
/swallow-pricing-engine:docs
```

Loads the full Swallow Pricing Engine documentation into context — step types, expression syntax, schema validation, templates, and more.

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

## MCP Tools

| Tool | Description |
|------|-------------|
| `test_swallow_project` | Run pricing engine against project tests |
| `validate_swallow_project` | Validate project JSON against schema |
| `schema_swallow_project` | Fetch full project JSON schema |

## Plugin Structure

```
swallow-claude-plugin/
├── .claude-plugin/
│   ├── plugin.json            # Plugin manifest
│   └── marketplace.json       # Marketplace listing
├── .mcp.json                  # MCP server connection
├── commands/
│   └── docs.md                # Full engine documentation
├── skills/
│   ├── build-pricing-model/   # Build from description
│   ├── excel-to-swallow/      # Convert Excel rater to Swallow
│   ├── validate-project/      # Schema validation
│   └── test-project/          # Run and analyse tests
├── agents/
│   ├── swallow-actuary.md     # Technical pricing builder
│   └── swallow-analyst.md     # Commercial pricing analyst
└── README.md
```

## API Endpoint

The plugin connects to `https://api.llow.io/ai/mcp`. For self-hosted deployments, update the URL in `.mcp.json`.

## License

MIT — Llow Group
