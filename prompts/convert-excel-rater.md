# convert-excel-rater

MCP prompt for converting an Excel rating workbook into a Swallow pricing engine configuration.

## Prompt definition

```json
{
  "name": "convert-excel-rater",
  "description": "Convert an Excel rating workbook (.xls/.xlsx) into a Swallow pricing engine JSON configuration by tracing formulas and dependencies",
  "arguments": [
    {
      "name": "workbook_description",
      "description": "Description of the Excel workbook structure — sheets, key cells, the result cell if known, and any special logic. Paste cell formulas and values if possible.",
      "required": true
    }
  ]
}
```

## Prompt content

You are an insurance pricing specialist converting an Excel rating workbook into a Swallow pricing engine configuration.

{workbook_description}

### Step 1 — Understand the workbook

From the description, identify:
- All cells with formulas (calculations)
- All cells with static values (potential inputs or constants)
- Named ranges and their references
- Cross-sheet references
- Any flagged cells (`:::result`, `:::input`, `:::valid` in comments)

### Step 2 — Identify the result cell

Find the result cell using this priority:

1. **Supplied by user** — if a cell reference was provided, use it
2. **`:::result` flag** — scan comments for `:::result`
3. **Auto-detect** — find terminal formula cells:
   a. Build a reference map: for every formula cell, record which cells it references
   b. Invert to find which cells are referenced BY other cells
   c. Terminal formula cells have a formula but are NOT referenced by any other formula
   d. Rank candidates by: deepest dependency chain, numeric type, sheet name heuristics (summary, output, premium, result, quote), cell label heuristics (total, premium, price, result, final, gross, net)
   e. Single clear winner → use it. Multiple candidates → present them to the user
4. **Ask the user** — last resort

Also look for:
- `:::valid` — the validity/exclusion check cell
- Terminal boolean formula cells → likely validity checks

### Step 3 — Trace the dependency tree

From the result cell, recursively trace every cell reference:
- Formula cells → recurse into their references
- Static value cells → leaf node (input or constant)
- Continue until all leaf cells are found

### Step 4 — Classify each cell

| Classification | Criteria | Swallow mapping |
|---|---|---|
| **Input** | Leaf cell, value changes per quote | `input.format` field |
| **Constant** | Leaf cell, fixed value (rates, thresholds) | `input.format` with `static: true` |
| **Lookup table** | Range used by VLOOKUP/INDEX/MATCH | `collection` step |
| **Calculation** | Arithmetic formula | `calculation` or `transform` step |
| **Conditional** | IF/AND/OR determining validity | `exclusion` or `refer` step |
| **Result** | Final price cell | `output.result` |
| **Valid** | Validity check cell | `output.valid` |

### Step 5 — Order into Swallow steps

Arrange by dependency depth:
1. **Inputs** (depth 0) — all leaf cells
2. **Collections** (depth 1) — lookup tables
3. **Transforms** (depth 2+) — intermediate calculations
4. **Exclusions/Refers** — conditional rules
5. **Final calculations** — premium, commission, tax
6. **Output** — result formula and valid expression

### Step 6 — Convert formulas to Swallow expressions

| Excel | Swallow |
|---|---|
| `=A1 * B1` | `({{input_a}} * {{input_b}})` |
| `=ROUND(A1, 2)` | `round({{input_a}}, 2)` |
| `=IF(A1>50, 1.5, 1.0)` | `({{age}} > 50 ? 1.5 : 1.0)` |
| `=VLOOKUP(A1, range, 3, FALSE)` | Collection: `collection.filter(col=={{lookup_val}}).map(value).first()` |
| `=VLOOKUP(A1, range, 3, TRUE)` | Collection: `collection.filter(min<={{val}}).filter(max>={{val}}).map(rate).first()` |
| `=INDEX(MATCH(...))` | Collection with filter/map |
| `=SUM(A1:A10)` | `_.sum([{{a1}}, {{a2}}, ...])` or collection `.map(value).sum()` |
| `=AND(A1>18, A1<80)` | Exclusion: `({{age}} < 18 \|\| {{age}} > 80)` (inverted — triggers on FAIL) |
| `=MIN(A1, B1)` | Calculation step: `min({{a}}, {{b}})` (NOT supported in transform `exp`) |
| `=MAX(A1, B1)` | Calculation step: `max({{a}}, {{b}})` (NOT supported in transform `exp`) |

Key rules:
- Replace all cell references with `{{swallow_property_name}}`
- Name properties after what they represent (`driver_age` not `sheet1_c5`)
- `id` and `key` must be the same within each step
- Every property needs a `def` — use the cell's current value
- In calculation `formula` fields: use `and`/`or` not `&&`/`||` (MathJS syntax)
- In exclusion/refer `exp` fields: `&&` and `||` ARE supported

### Step 7 — Build the Swallow JSON

Assemble the complete project. Every input field MUST have `key`, `exp`, `label`, `sub_label`, `type`, and `def`. Without `key` and `exp`, the engine ignores quote overrides and always uses `def`.

Every step MUST have `id`, `step`, `name`, `key`, and `description`.

Collection data MUST be in compressed format: `[[headers], [row1], [row2], ...]`.

```json
{
  "meta": { "name": "...", "description": "Converted from [filename]", "summary": "2-4 sentences: what the model rates, jurisdiction, methodology, audience." },
  "input": { "format": {
    "example_field": {
      "key": "example_field",
      "label": "Example",
      "sub_label": "At least 20 characters, written for brokers/customers.",
      "type": "string",
      "def": "default_value",
      "exp": "{{example_field}}"
    }
  } },
  "steps": [ ],
  "output": {
    "result": { "key": "result", "formula": "round({{total_premium}}, 2)", "type": "decimal", "def": 0 },
    "valid": { "key": "output", "exp": "{{exclusions.count()}} == 0", "type": "boolean", "def": true },
    "format": { }
  },
  "tests": [ ]
}
```

`output.format` is mandatory — expose per-coverage premiums and key intermediate values.

### Step 8 — Build test cases from the spreadsheet

The workbook's current values are a built-in test case:
- **Input**: current values of all input cells
- **Expected output**: current value of the result cell

Every test must have `output` with `result` (number) and `valid` (boolean) — never `{}`.

Add 2-3 more test cases by varying key inputs.

ALL tests must pass — zero exceptions. Never set `expected.result` to match engine output to force a pass.

### Step 9 — Validate and test

1. Call `validate_swallow_project` — fix any schema errors
2. Call `test_swallow_project` — check pricing matches the Excel
3. If results don't match, read `result.debug.steps` to trace intermediate values. Find the FIRST step where values diverge.
4. Iterate until tests pass

### Step 10 — Report

Summarise:
- Number of inputs, steps, and outputs
- Test results (expected vs actual)
- Any formulas that couldn't be automatically translated (flag for manual review)

## Rules

Follow these rules in all generated configurations:

### Naming
- `id` and `key` must be identical within a step, unique across the project
- Never use a step type name or its plural as a key — e.g. never `"exclusion"`, `"exclusions"`, `"refer"`, `"refers"`. Use prefixed names: `"uw_exclusions"`, `"risk_refers"`
- Never use `key` as a data column name in collections (reserved by engine)
- Use `snake_case` for all keys and property names

### Expressions
- Wrap all property references in `{{double_curly_braces}}`
- Use `==` not `===`, `!=` not `!==`
- In calculation `formula` fields: use `and` not `&&`, use `or` not `||` (MathJS syntax)
- In exclusion/refer `exp` fields: `&&` and `||` ARE supported
- `min()`/`max()` not supported in transform `exp` — use calculation steps
- Use `round()` for monetary values
- Use `_.` prefix for lodash methods

### Default values
- Multiplicative factor defaults must be `1.0` — a `0` default silently zeros the entire premium chain
- Additive component defaults should be `0`
- Boolean flag defaults: `false`

### Steps
- Every step must have `id`, `step`, `name`, `key`, `description`
- Every `transform` must have `retain: true` — without it, subsequent steps lose ALL prior values
- Collection data must be in compressed format: `[[headers], [row1], ...]`
- Endorsement `def` must be `[]`
- Use: `transform`, `exclusion`, `refer`, `excesses`, `endorsements`, `calculation`, `collection`, `external`, `modules`
- Never use: `factors`, `modular`, `batch`, `links`, `label`, `code`

### General
- No comments in JSON
- No nested input paths
- Input types: `string`, `number`, `integer`, `decimal`, `boolean`, `date` only
- Valid JSON only
