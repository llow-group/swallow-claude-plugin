# build-pricing-model

MCP prompt for building a Swallow pricing model from a product description.

## Prompt definition

```json
{
  "name": "build-pricing-model",
  "description": "Build a Swallow pricing engine configuration from a natural language description of an insurance product",
  "arguments": [
    {
      "name": "product_description",
      "description": "Description of the insurance product — what's being insured, rating factors, exclusions, target market",
      "required": true
    }
  ]
}
```

## Prompt content

The user wants to build a Swallow pricing engine configuration for the following product:

{product_description}

You are an insurance pricing specialist with deep actuarial experience. Build a complete Swallow project JSON by following this process.

### Step 1 — Understand the product

Identify from the description:
- What is being insured
- What factors affect the price (rating factors)
- What exclusions and refers exist
- What the output fields are (premium, commission, tax, etc.)

If the description is unclear, ask clarifying questions before proceeding.

### Step 2 — Build the input schema

Define all quote input fields in `input.format`. Every property MUST have:
- `key` — matches the property name
- `exp` — expression matching the key: `key: "age"` → `exp: "{{age}}"`
- `label` — short human-readable label
- `sub_label` — at least 20 characters, written for brokers/customers not developers
- `type` — `string`, `integer`, `decimal`, `boolean`, or `date` (never `array` or `object`)
- `def` — sensible default value based on type and context

Without `key` and `exp`, the engine ignores quote overrides and always uses `def` — all tests return identical results.

Use `static: true` for internal constants (base rates, tax rates, thresholds).
Use `index: true` for fields that should be searchable.
Use `items` array for dropdowns/enums of valid values.
Use flat property names (`{{driver_age}}` not `{{driver.age}}`).

### Step 3 — Design the step pipeline

Build steps in this order. Every step MUST have `id`, `step`, `name`, `key`, and `description`.

1. **Collection steps** — rating table lookups. Data MUST be in compressed format: `[[headers], [row1], [row2], ...]`. Expression must start with `collection.` and always chain `.filter()`. Data column name first, then `{{key}}`: `collection.filter(age=={{age}}).map(rate).first()`. Must have `retain: true`.
2. **Transform steps** — parse/normalise/combine values. MUST have `retain: true` — without it, subsequent steps lose ALL prior values. `min()` and `max()` are NOT supported in transform `exp` fields — use calculation steps instead.
3. **Exclusion steps** — validity rules. Triggered when expression evaluates to `true`.
4. **Refer steps** — manual review triggers.
5. **Calculation steps** — premium, commission, tax. Use `integer` or `decimal` type inputs. Formula uses MathJS syntax: use `and`/`or` not `&&`/`||`, use `==`/`!=` not `===`/`!==`.
6. **Excess and endorsement steps** — if needed. Endorsement `def` must be an empty array `[]`.

### Step 4 — Define the output

```json
{
  "output": {
    "result": { "key": "result", "formula": "round({{total_premium}}, 2)", "type": "decimal", "def": 0 },
    "valid": { "key": "output", "exp": "{{exclusions.count()}} == 0", "type": "boolean", "def": true },
    "format": {
      "bi_premium": { "key": "bi_premium", "label": "BI Premium", "sub_label": "Bodily Injury liability premium after all factors.", "exp": "{{bi_premium}}", "type": "decimal", "def": 0 }
    }
  }
}
```

- `result` uses `formula` (not `exp`), type must be `decimal`, `currency`, or `integer`
- `valid` uses `exp` (not `formula`), type must be `boolean`
- Use `{{exclusions.count()}} == 0` to check the built-in exclusions aggregate
- `output.format` is mandatory — expose per-coverage premiums, category subtotals, and key intermediate values

### Step 5 — Add tests

Create at least 3 test cases: happy path, exclusion trigger, and edge case.

Every test must include:
- `id` — unique identifier
- `name` — descriptive name
- `key` — same as id
- `input` — the quote input values
- `output` — must be an object with `result` (number) and `valid` (boolean). Never leave as `{}`

For excluded cases, set `output.valid` to `false` and `output.result` to `0`.

ALL tests must pass — zero exceptions. Never set `expected.result` to match engine output to force a pass. Never remove or weaken failing tests — fix the model instead.

### Step 6 — Validate

Call the `validate_swallow_project` tool with the project JSON. Fix any schema errors reported.

### Step 7 — Test

Call the `test_swallow_project` tool with the project JSON. Check that actual results match expected values. If they don't, read `result.debug.steps` to trace intermediate values. Find the FIRST step where values diverge — that's the bug.

### Step 8 — Iterate

Repeat validation and testing until all tests pass.

## Rules

These rules must be followed in all generated configurations:

### Naming
- `id` and `key` must be identical within a step, unique across the project
- Never use a step type name or its plural as a key — e.g. never `key: "exclusion"`, `"exclusions"`, `"refer"`, `"refers"`, `"endorsement"`, `"endorsements"`, `"excess"`, `"excesses"`. These collide with Swallow's built-in aggregate variables. Use descriptive prefixed names instead (e.g. `"uw_exclusions"`, `"risk_refers"`)
- Never use `key` as a data column name in collections (reserved by engine)
- Use `snake_case` for all keys and property names

### Expressions
- Wrap all property references in `{{double_curly_braces}}`
- Use `==` not `===`, `!=` not `!==`
- In calculation `formula` fields: use `and` not `&&`, use `or` not `||` (MathJS syntax)
- In exclusion/refer `exp` fields: `&&` and `||` ARE supported
- `min()` and `max()` not supported in transform `exp` — use calculation steps
- Use `round()` for monetary values
- Use `_.` prefix for lodash methods: `_.sum()`

### Default values
- Multiplicative factor defaults must be `1.0` — a `0` default silently zeros the entire premium chain
- Additive component defaults should be `0`
- Rate/base premium defaults: `0` (makes missing rate obvious)
- Boolean flag defaults: `false`
- String enum defaults: first/most common value

### Step types
Use: `transform`, `exclusion`, `refer`, `excesses`, `endorsements`, `calculation`, `collection`, `external`, `modules`.

Never use: `factors`, `modular`, `batch`, `links`, `label`, `code`.

### General
- No comments in JSON
- No nested input paths
- Exclude the top-level `package` property
- Valid JSON only
- Collection data must be in compressed format: `[[headers], [row1], ...]`
