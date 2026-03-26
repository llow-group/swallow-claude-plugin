# Swallow Agent Rules

Rules and constraints for AI agents building, validating, and testing Swallow pricing models via the MCP server or Claude Chat.

---

## 1. Config Structure

Every Swallow config has four top-level sections: `meta`, `input`, `steps[]`, `output`.

### Meta

```json
{
  "meta": {
    "name": "Product Name",
    "description": "Short description",
    "summary": "2-4 sentence paragraph: what the model rates, jurisdiction, methodology, audience.",
    "project_reference": "uuid",
    "updated_at": "2024-01-01T00:00:00Z"
  }
}
```

### Input Format

Every input field MUST have: `key`, `exp`, `label`, `sub_label`, `type`, `def`.

```json
{
  "key": "zip_code",
  "label": "ZIP Code",
  "sub_label": "5-digit US ZIP code for the insured location. Determines territory rating and base rates. Find this on any piece of mail addressed to the property.",
  "type": "string",
  "def": "90001",
  "exp": "{{zip_code}}"
}
```

**Critical rules:**
- Without `key` and `exp`, the engine ignores quote overrides and always uses `def` — all tests return identical results
- `exp` must match `key`: `key: "zip_code"` -> `exp: "{{zip_code}}"`
- Use flat property names: `{{address_postcode}}` NOT `{{address.postcode}}`
- Input types: `string`, `number`, `integer`, `decimal`, `boolean`, `date` only (never `array` or `object` at top level, except for module parent inputs)
- `sub_label` minimum 20 characters, written for brokers/customers, not developers
- `items` array for dropdowns/enums of valid values
- `def` must be a reasonable value based on type and context

### Steps

Steps are an ordered array. Each step receives accumulated results of all prior steps.

**Allowed step types for auto-generated models:** `collection`, `calculation`, `transform`, `modules`, `exclusion`, `refer`, `endorsements`, `excesses`

**NEVER use:** `factors`, `modular`, `batch`, `links`, `label`, `code` steps in generated configs.

#### Collection Steps

Lookup tables in compressed format.

```json
{
  "id": "territory_rates",
  "step": "collection",
  "name": "Territory Rate Lookup",
  "key": "territory_rates",
  "description": "Looks up base rate by ZIP code. 200 rows covering all rated territories.",
  "retain": true,
  "collection": [
    ["zip", "territory", "bi_rate", "pd_rate"],
    ["90001", "LA", 247.00, 189.50],
    ["10001", "NYC", 312.00, 245.80]
  ],
  "format": {
    "bi_base_rate": {
      "key": "bi_base_rate",
      "label": "BI Base Rate",
      "exp": "collection.filter(zip=={{zip_code}}).map(bi_rate).first()",
      "type": "decimal",
      "def": 0
    }
  }
}
```

**Collection rules:**
- Data MUST be in compressed format: `[[headers], [row1], [row2], ...]`
- `exp` MUST start with `collection.` — never use other transform logic
- Chain `.filter()` for lookups: `collection.filter(col=={{input}}).map(value).first()`
- NEVER use `key` as a data column name (reserved by engine)
- MUST have `retain: true`
- Filter patterns: exact `=={{ref}}`, range `<={{ref}}` / `>={{ref}}`, multi-key chained `.filter().filter()`
- Aggregation: `.sum(col)`, `.count()`, `.map(col).first()`, `.map(col).last()`
- When filter matches no rows, `def` is used — set multiplicative factor defaults to `1.0`, not `0`

#### Calculation Steps

MathJS formulas. Variables referenced as `{{variable_name}}`.

```json
{
  "id": "calc_bi_premium",
  "step": "calculation",
  "name": "BI Premium",
  "key": "bi_premium",
  "description": "Multiplies BI base rate by 6 rating factors, rounds to nearest cent.",
  "def": 0,
  "formula": "round({{bi_base_rate}} * {{territory_factor}} * {{age_factor}} * {{class_factor}}, 2)"
}
```

**Calculation rules:**
- Use `==` NOT `===` for equality
- Use `!=` NOT `!==` for inequality
- Use `and` NOT `&&` for logical AND
- Use `or` NOT `||` for logical OR
- All variable references as `{{variable_name}}`
- Calculations always retain previous data
- Unresolved `{{variable}}` in multiplication defaults to `0`, zeroing the entire chain

#### Transform Steps

Derived values. MUST have `retain: true`.

```json
{
  "id": "derived_values",
  "step": "transform",
  "name": "Derived Values",
  "key": "derived_values",
  "description": "Computes adjusted limit and active endorsement flags.",
  "retain": true,
  "format": {
    "adjusted_limit": {
      "key": "adjusted_limit",
      "label": "Adjusted Limit",
      "exp": "{{coverage_a_limit}} * {{rc_adjustment}}",
      "type": "decimal",
      "def": 0
    }
  }
}
```

**Transform rules:**
- `retain: true` is MANDATORY — without it, subsequent steps lose ALL prior values
- `min()` and `max()` are NOT supported in transform `exp` fields — use calculation steps instead
- `round()` works in transforms

#### Module Steps

For multi-item rating (multiple vehicles, travelers, locations). NEVER use `code` steps.

```json
{
  "step": "modules",
  "name": "Per-Traveler Rating",
  "key": "traveler_rating",
  "options": {
    "root_property": "travelers",
    "debug": false
  },
  "format": {
    "total_premium": {
      "label": "Total Premium",
      "exp": "collection.sum(result)",
      "type": "decimal",
      "def": 0
    }
  }
}
```

The child model goes in `config.modules[]` with `root_property` matching the parent step.

#### Rule Steps (Exclusion, Refer, Endorsements, Excesses)

```json
{
  "id": "my_exclusions",
  "step": "exclusion",
  "key": "my_exclusions",
  "items": [
    { "id": "too_young", "name": "Too young", "exp": "{{age}} < 17", "def": false }
  ]
}
```

- Exclusion/refer step `id` and `key` must NOT be named `exclusion`, `refer`, `exclusions`, or `refers`
- Endorsement step `def` must be an array (empty or otherwise)
- A step's `id` and `key` must be the same value, but unique across the entire config

### Output

```json
{
  "output": {
    "result": {
      "key": "result",
      "formula": "round({{total_premium}}, 2)",
      "type": "decimal",
      "def": 0
    },
    "valid": {
      "key": "output",
      "exp": "{{exclusions.count()}} == 0",
      "def": true,
      "type": "boolean"
    },
    "format": {
      "bi_premium": {
        "key": "bi_premium",
        "label": "BI Premium",
        "sub_label": "Bodily Injury liability premium after all factors.",
        "exp": "{{bi_premium}}",
        "type": "decimal",
        "def": 0
      }
    }
  }
}
```

**Output format is mandatory.** Must expose: per-coverage premiums, category subtotals, aggregation bases, total modifier, classification base.

### Tests

```json
{
  "tests": [
    {
      "id": "source_test_1",
      "key": "example_1",
      "name": "Filing Example 1",
      "category": "source",
      "input": { "zip_code": "90036", "bi_limit": "30_60" },
      "output": { "result": 3471.25 },
      "expected": { "result": 3471.25, "valid": true }
    }
  ]
}
```

---

## 2. Expression Syntax

The engine uses MathJS for formulas and Swallow QL for property access. This is NOT JavaScript.

| Use | NOT | Context |
|-----|-----|---------|
| `==` | `===` | Equality |
| `!=` | `!==` | Inequality |
| `and` | `&&` | Logical AND (formulas) |
| `or` | `\|\|` | Logical OR (formulas) |
| `{{variable}}` | bare `variable` | Variable references |

**Note:** In `exp` fields for exclusion/refer/label items (non-formula contexts), `&&` and `||` ARE supported. The MathJS restriction applies to `formula` fields in calculation steps and format `exp` fields.

### Lodash

Use `_.` prefix for lodash methods: `_.sum()`, `_.mean()`, etc.

### Collection Query Language

```
collection.filter(column=={{input_ref}}).map(value_col).first()
collection.filter(col1=={{ref1}}).filter(col2=={{ref2}}).map(value).first()
collection.filter(min<={{value}}).filter(max>={{value}}).map(factor).first()
collection.map(result).sum()
collection.sum(premium)
collection.count()
```

### Ternary Conditionals

```
{{is_good_driver}} == true ? 0.80 : 1.00
({{driving_record_points}} >= 5) ? {{surcharge}} : 1.00
({{cdw_limit}} != 'no_coverage') ? 1.05 : 1.00
```

### Unresolved References

| Context | Behaviour | Danger |
|---------|-----------|--------|
| Multiplication formula | Defaults to `0` | **Zeros the entire chain** |
| Addition formula | Defaults to `0` | Silently drops the term |
| Collection filter | No rows match | Falls back to `def` |
| Transform exp | Expression fails | Falls back to `def` |

### Default Value Strategy

| Usage | Safe default | Why |
|-------|-------------|-----|
| Multiplicative factor | `1.0` | Neutral multiplier |
| Additive component | `0` | Neutral addend |
| Rate/base premium | `0` | Makes missing rate obvious |
| Boolean flag | `false` | Conservative |
| String enum | First/most common value | Valid lookup result |

**Never use `0` as default for a multiplicative factor** — it silently zeros the entire premium chain.

---

## 3. Absolute Rules

### NEVER

- Use `factors`, `modular`, `batch`, `links`, `label`, or `code` steps
- Give a step an `id` or `key` that matches a reserved step name (`exclusion`, `refer`, `exclusions`, `refers`)
- Use nested input paths (`{{address.postcode}}`); use flat names (`{{address_postcode}}`)
- Set input type to `array` or `object` (must be `string`, `number`, `integer`, `decimal`, or `boolean`)
- Use `===`, `!==`, `&&`, `||` in calculation formulas
- Leave comments in config JSON
- Include the top-level `package` property
- Use `key` as a data column name in collections
- Use `min()`/`max()` in transform `exp` fields
- Hardcode test answers into formulas (e.g., `({{x}} == 690) ? 2189.67 : {{calc}}`)
- Change `expected.result` to match engine output — fix the model instead
- Remove or weaken failing tests
- Leave collection steps with empty data or all-1.0 placeholder values
- Reverse-engineer base rates from test answers
- Build a model for a different product than the source describes
- Skip `key` or `exp` on input format fields
- Use `min()` or `max()` in transform steps

### ALWAYS

- Use `_.` prefix for lodash methods
- Wrap internal property references in `{{}}`
- Use compressed collection format: `[[headers], [row1], ...]`
- Start collection step `exp` with `collection.` and use only QL syntax
- Chain `.filter()` in collection steps (never use other logic)
- Have every transform step set `retain: true`
- Make `id` and `key` the same within a step, but unique across the config
- Have a `def` value on every property — best guess based on type/context/enum
- Use `integer` or `decimal` type properties in calculation steps
- Have endorsement step `def` as an array
- Match `exp` to input `key`: `key: "zip_code"` -> `exp: "{{zip_code}}"`
- Use `==` (loose equality) instead of `===`
- Validate JSON before submitting
- Include `description` on every step
- Include `sub_label` (min 20 chars) on every input
- Include `output.format` with per-coverage premiums

---

## 4. Building from Source

### From SERFF Filing (PDF)

**Pipeline: Extract -> Build -> Validate -> Spreadsheet**

1. **Extract** all 13 structured files from the filing (12 JSON + `extraction_summary.md`)
2. **Build** the Swallow config using the product-specific generator
3. **Validate** — ALL tests must pass (zero exceptions, penny-level accuracy)
4. **Generate spreadsheets** — both Excel and Google Sheets format

**Buildability check — ALL must be true:**
- `coverages.json` has >= 1 coverage with non-null `base_rate`
- `rates_data.json` has >= 1 table with data rows
- `calculations.json` has non-empty `step_order`
- `examples.json` has >= 1 example with `total_premium`
- `final_rating_calculation.json` has non-empty `algorithm.steps`
- `extraction_summary.md` exists with `viability = BUILDABLE`

### From XLS Rater (Spreadsheet)

**Pipeline: Ingest -> Extract -> Build -> Validate -> Spreadsheet**

1. **Ingest** XLS/XLSX files to produce `normalised_xls.json`
2. **Find `:::result` cell** — the cell comment marking the final premium output
3. **Trace backwards** from result cell to reconstruct the rating algorithm
4. **Extract** the same 12 structured files
5. **Build and validate** as above

**The `:::result` cell gives you:**
- The target value (ground truth premium)
- The formula chain (trace dependencies backward)
- Input values (cells feeding into the chain)

### From User Prompt (MCP/Chat)

When a user describes a rating model conversationally:
1. Identify the product type, coverages, rating factors, and any rate tables
2. Build the config following all rules above
3. Use the `validate_swallow_project` MCP tool to check the config
4. Use the `test_swallow_project` MCP tool to run tests
5. Iterate until all tests pass

---

## 5. Quality Gates

### Test Requirements

- **ALL tests must pass — zero exceptions**
- At least 1 test must have `category: "source"` from a filing worked example
- Source test `expected.result` must match filing to penny-level accuracy
- Synthetic tests alone are insufficient — they only prove internal consistency
- NEVER set `expected.result` to match engine output to force a "PASS"
- NEVER remove failing tests — fix the model instead
- NEVER adjust source test inputs to make them pass
- A model with ANY failing test is BROKEN, not "close enough"

### Validation Checks

1. **Expression integrity** — no NaN, undefined, Infinity, empty expressions, unresolved templates, unbalanced brackets
2. **Reference integrity** — every `{{variable}}` resolves to an input key or preceding step output
3. **Collection completeness** — every rate table from source has real data, not placeholders
4. **Algorithm completeness** — model step count matches filing step count
5. **Product identity** — model name/description matches the actual filing product
6. **Factor variation** — different inputs produce different premiums
7. **Schema compliance** — every input has `label`, `type`, `def`; every step has `id`, `step`, `name`, `key`

### Cohort Analysis (2,500 tests)

After passing validation, generate cohort tests to check:
- No negative premiums
- No zero premiums for valid quotes
- No unreasonably high/low premiums vs industry norms
- Price distribution shape matches product expectations
- Every rated input produces premium variation
- Factor sensitivity — changing one input changes the output

---

## 6. Actuarial Principles

- **Base rates come from the filing**, not reverse-engineered from examples
- **Factor tables must be complete** — all rows from filing, never sampled or truncated
- **The rating algorithm is sacred** — every step in the filing needs a model step
- **Worked examples are ground truth** — if engine disagrees, the engine is wrong
- **Rounding matters** — premiums must match to the penny
- **Rate ranges require deterministic scoring** — convert to scored inputs with pre-computed interpolated rates
- **Minimum premium floors** must be enforced when specified by filing
- **Coverage dependencies** must be modeled as exclusion rules (e.g., GAP requires COMP+COLL)
- **Expense constants and fees** (policy fees, TRIA surcharges) must be included if filed

---

## 7. Debugging

When tests fail:
1. Read `result.debug.steps` to trace intermediate values
2. Compare against the filing's worked example
3. Find the FIRST step where values diverge — that's the bug
4. Common causes:
   - Missing `key`/`exp` on inputs (engine uses defaults for everything)
   - Missing `retain: true` on transforms (downstream steps lose all values)
   - `===` or `&&` in MathJS formulas (silent wrong evaluation)
   - Unresolved `{{variable}}` in multiplication (silently becomes 0)
   - Collection with no matching rows falling back to `def: 0` in a multiplication
   - Wrong column name in collection filter
   - Missing rate table data (empty collection)

---

## 8. MCP Tool Usage

### `schema_swallow_project`
Returns the full JSON schema for Swallow configs. Use when building a new config to verify structure.

### `validate_swallow_project`
Validates a config against the schema and checks expression integrity. Run after every edit.

### `test_swallow_project`
Runs embedded tests through the pricing engine. ALL tests must pass before the model is considered complete.

---

## 9. Common Patterns

### Multiplicative Factor Chain
```
base_rate * territory * age * class * experience * schedule = premium
```

### Minimum Premium Floor
```
"formula": "max({{calculated_premium}}, {{minimum_premium}})"
```

### Coverage Conditional
```
"formula": "{{comp_limit}} != 'no_coverage' ? round({{comp_base}} * {{comp_factor}}, 2) : 0"
```

### Multi-Coverage Total
```
"formula": "round({{bi_premium}} + {{pd_premium}} + {{comp_premium}} + {{coll_premium}}, 2)"
```

### Exposure Scoring (for rate ranges)
1. Create scored inputs per dimension (1-5 scale)
2. Aggregate to total score via calculation step
3. Pre-compute interpolated rates for each valid score
4. Emit collection lookups keyed by score
5. Include override input for direct rate entry
