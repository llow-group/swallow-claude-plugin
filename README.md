# Swallow Pricing Engine Plugin

A Claude Code plugin that connects to the [Swallow](https://swallow.app) insurance pricing engine via MCP. Build, validate, analyse, and test insurance pricing models directly from Claude.

## What you get

- **MCP Server** — 3 tools: test project, validate project, fetch schema
- **Skills** — `/build-pricing-model`, `/validate-project`, `/test-project`
- **Agents** — `swallow-actuary` (builds models) and `swallow-analyst` (reviews and stress-tests them)
- **Context** — Full engine documentation and LLM generation guidelines loaded as rules

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
/build-pricing-model

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

### Validate an existing project

```
/validate-project

Check the project in ./my-project.json
```

### Run tests

```
/test-project

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
│   ├── plugin.json                # Plugin manifest
│   └── marketplace.json           # Marketplace listing
├── .mcp.json                      # MCP server connection
├── .claude/rules/
│   ├── swallow-readme.md          # Full engine documentation
│   └── swallow-llm-guidelines.md  # LLM generation rules
├── skills/
│   ├── build-pricing-model/       # Build from description
│   ├── validate-project/          # Schema validation
│   └── test-project/              # Run and analyse tests
├── agents/
│   ├── swallow-actuary.md         # Technical pricing builder
│   └── swallow-analyst.md         # Commercial pricing analyst
└── README.md
```

## Adding your own documentation

Drop additional `.md` files into `.claude/rules/` to give Claude more context about your specific insurance products, rating tables, or conventions. These are loaded automatically.

## API Endpoint

The plugin connects to `https://api.llow.io/ai/mcp`. For self-hosted deployments, configure the endpoint in plugin settings.

## License

Proprietary — Llow Group
