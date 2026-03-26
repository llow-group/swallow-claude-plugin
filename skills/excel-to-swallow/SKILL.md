---
name: excel-to-swallow
description: Convert an Excel rating workbook (.xls/.xlsx) into a Swallow pricing engine configuration. Reads formulas, traces dependencies, and rebuilds as Swallow JSON.
tools:
  - MCP
  - Read
  - Write
  - Bash
---

# Excel to Swallow Converter

The user wants to convert an Excel rating workbook into a Swallow pricing engine configuration.

## Inputs

The user will provide:
- **Excel file path** — a `.xls` or `.xlsx` file containing pricing/rating logic
- **Result cell** (optional) — the cell reference containing the final price (e.g. `"Sheet1!C45"` or `"premium!H12"`)

If no result cell is supplied, **auto-detect it** (see Step 2). Only ask the user as a last resort.

## Process

### Step 1 — Read and understand the workbook

Read the Excel file. For each sheet, map out:
- All cells with formulas (these are calculations)
- All cells with static values (these are potential inputs or constants)
- All cells with comments/notes (check for `:::result`, `:::input`, or `:::valid` flags)
- Named ranges and their cell references
- Cross-sheet references

### Step 2 — Identify the result cell

Find the result cell using this priority:

1. **Supplied parameter** — if the user provided a cell reference, use it
2. **`:::result` flag** — scan all cell comments for `:::result`
3. **Auto-detect** — this is the primary method. Find the result cell by:
   a. Build a reference map: for every cell with a formula, record which other cells it references
   b. Invert the map to find which cells are referenced BY other cells
   c. **Terminal formula cells** are cells that have a formula but are NOT referenced by any other formula cell — they are the "end of the chain"
   d. Among terminal cells, rank candidates by:
      - Deepest dependency chain (most steps to reach from inputs) — the final price typically depends on many intermediate calculations
      - Numeric type (the result is almost always a number, not text or boolean)
      - Sheet name heuristics: cells on sheets named "summary", "output", "premium", "result", "quote" rank higher
      - Cell label/header heuristics: cells near labels containing "total", "premium", "price", "result", "final", "gross", "net" rank higher
   e. If there is a single clear winner, use it. If there are 2-3 plausible candidates, present them to the user with their current values and formula descriptions and ask which is the result
4. **Ask the user** — only as last resort if auto-detect produces too many candidates or none

Also look for:
- `:::valid` in comments — the validity/exclusion check cell
- `:::input` in comments — explicitly marked input cells
- Auto-detect validity: terminal boolean formula cells that reference exclusion-like logic

### Step 3 — Trace the dependency tree backwards from result

Starting from the result cell, recursively trace every cell reference in its formula:
- For each referenced cell, check if it has a formula (calculation) or a static value (input/constant)
- For cells with formulas, recurse into their references
- Continue until you reach leaf cells (no formula, no further references)

Build a dependency map:
```
result_cell
  ├── calc_cell_1 (formula: =A1 * B1)
  │   ├── input_cell_A1 (static value: 500)
  │   └── input_cell_B1 (static value: 1.2)
  ├── calc_cell_2 (formula: =VLOOKUP(...))
  │   ├── lookup_table (range of static data)
  │   └── input_cell_C3 (static value: "SW1")
  └── ...
```

### Step 4 — Classify each cell

For every cell in the dependency tree, classify it as:

| Classification | Criteria | Swallow mapping |
|---|---|---|
| **Input** | Leaf cell, no formula, value would change per quote | `input.format` field |
| **Constant** | Leaf cell, no formula, value is fixed (rates, thresholds) | `input.format` field with `static: true` |
| **Lookup table** | Range of static data used by VLOOKUP/INDEX/MATCH | `collection` step |
| **Calculation** | Cell with arithmetic formula | `calculation` step or `transform` step |
| **Conditional** | Cell with IF/AND/OR that determines validity | `exclusion` or `refer` step |
| **Result** | The final price cell | `output.result` |
| **Valid** | The validity check cell | `output.valid` |

When classifying, consider:
- Cells that look like customer data (age, postcode, name, vehicle value) → **Input**
- Cells that look like fixed rates or thresholds (tax rate, commission %, loading factors) → **Constant** (static input)
- Ranges used in VLOOKUP/INDEX/MATCH → **Lookup table** (collection)
- Boolean cells that gate whether a quote is valid → **Conditional** (exclusion)

### Step 5 — Order into Swallow steps

Arrange the cells into an ordered pipeline. Use the dependency depth:
1. **Inputs** (depth 0) — all leaf cells
2. **Collections** (depth 1) — lookup tables
3. **Transforms** (depth 2+) — intermediate calculations, grouped by sheet or logical concern
4. **Exclusions/Refers** — conditional rules
5. **Final calculations** — premium, commission, tax
6. **Output** — result formula and valid expression

### Step 6 — Convert formulas to Swallow expressions

Translate Excel formulas to Swallow syntax:

| Excel | Swallow |
|---|---|
| `=A1 * B1` | `({{input_a}} * {{input_b}})` |
| `=ROUND(A1, 2)` | `round({{input_a}}, 2)` |
| `=IF(A1>50, 1.5, 1.0)` | `({{age}} > 50 ? 1.5 : 1.0)` |
| `=VLOOKUP(A1, Sheet2!A:C, 3, TRUE)` | Collection step: `collection.items.filter(key <= {{lookup_val}}).map(value).last()` |
| `=SUM(A1:A10)` | `_.sum([{{a1}}, {{a2}}, ...])` or collection `collection.items.map(value).sum()` |
| `=INDEX(MATCH(...))` | Collection step with filter/map |
| `=CONCATENATE(A1, B1)` | String template or code step |
| `=AND(A1>18, A1<80)` | Exclusion: `({{age}} < 18 \|\| {{age}} > 80)` (inverted — exclusion triggers on FAIL) |

Key rules:
- Replace all cell references with `{{swallow_property_name}}`
- Use `snake_case` for all property names (derived from cell labels, sheet names, or column headers)
- `id` and `key` must be the same within each step
- Every property needs a `def` (default value) — use the cell's current value. Multiplicative factor defaults must be `1.0` not `0`
- `result` uses `formula`, `valid` uses `exp`
- In calculation `formula` fields: use `and`/`or` not `&&`/`||` (MathJS syntax)
- `min()`/`max()` not supported in transform `exp` — use calculation steps
- Never use `key` as a data column name in collections
- Never use: `factors`, `modular`, `batch`, `links`, `label`, `code` steps

### Step 7 — Build the Swallow JSON

Every input field MUST have `key`, `exp`, `label`, `sub_label` (min 20 chars, broker-facing), `type`, and `def`. Without `key` and `exp`, the engine ignores quote overrides.

Every step MUST have `id`, `step`, `name`, `key`, and `description`.

Collection data MUST be in compressed format: `[[headers], [row1], [row2], ...]`.

Assemble the complete project:

```json
{
  "meta": { "name": "...", "description": "Converted from [filename]" },
  "input": { "format": { ... } },
  "steps": [ ... ],
  "output": {
    "result": { "key": "result", "formula": "...", "type": "decimal", "def": 0 },
    "valid": { "key": "output", "exp": "{{exclusions.count()}} == 0", "type": "boolean", "def": true },
    "format": { ... }
  },
  "tests": [ ... ]
}
```

### Step 8 — Build test cases from the spreadsheet's current values

The Excel file's current cell values are a built-in test case. Create a test using:
- **Input**: the current values of all input cells
- **Expected output**: the current value of the result cell

Add 2-3 more test cases by varying key inputs.

### Step 9 — Validate and test via MCP

1. Call `validate_swallow_project` — fix any schema errors
2. Call `test_swallow_project` — check the pricing matches the Excel
3. If the result doesn't match the Excel's value, trace the discrepancy:
   - Compare each intermediate step's output to the corresponding Excel cell
   - Fix the formula translation and re-test
4. Iterate until the test passes

### Step 10 — Write the output

Save the final Swallow JSON to a file and report:
- Number of inputs, steps, and outputs
- Test results (expected vs actual)
- Any formulas that couldn't be automatically translated (flag for manual review)

## Common Excel patterns and their Swallow equivalents

### VLOOKUP with exact match
```
Excel: =VLOOKUP(age, rate_table, 2, FALSE)
Swallow: collection step with filter(key == {{age}}).map(rate).first()
```

### VLOOKUP with approximate match (banded lookup)
```
Excel: =VLOOKUP(age, rate_table, 2, TRUE)
Swallow: collection step with filter(min_age <= {{age}}).filter(max_age >= {{age}}).map(rate).first()
Note: You may need to restructure the lookup table to have min/max columns
```

### INDEX/MATCH
```
Excel: =INDEX(rates, MATCH(postcode, postcodes, 0))
Swallow: collection step with filter(postcode == {{postcode}}).map(rate).first()
```

### Nested IF chains
```
Excel: =IF(age<25, 1.5, IF(age<50, 1.0, IF(age<65, 0.9, 1.2)))
Swallow: Either a transform with ternary chain, or better — a collection lookup table:
  [{"min":0,"max":24,"factor":1.5}, {"min":25,"max":49,"factor":1.0}, ...]
```

### SUMPRODUCT
```
Excel: =SUMPRODUCT(weights, values)
Swallow: Multiple calculation steps, or a code step for complex array operations
```

## Important notes

- Name properties after what they represent, not their cell reference (e.g. `driver_age` not `sheet1_c5`)
- Group related calculations into the same transform step when they're at the same dependency depth
- If a formula is too complex to translate cleanly, use a `code` step with JavaScript
- Always validate and test — the goal is for the Swallow model to produce the same price as the Excel for the same inputs
