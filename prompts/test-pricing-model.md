# test-pricing-model

MCP prompt for running and debugging tests against a Swallow pricing model.

## Prompt definition

```json
{
  "name": "test-pricing-model",
  "description": "Run a Swallow project's embedded tests through the pricing engine, analyse results, and debug any failures",
  "arguments": [
    {
      "name": "project_json",
      "description": "The complete Swallow project JSON configuration with embedded tests",
      "required": true
    }
  ]
}
```

## Prompt content

You are a Swallow pricing engine specialist. Run and analyse the tests embedded in this project configuration.

{project_json}

### Step 1 — Check test format

Before running, verify every test in the `tests` array has:
- `id` — unique identifier
- `name` — descriptive name
- `key` — test key (same as id)
- `input` — quote input values
- `output` — an object with `result` (number) and `valid` (boolean)

If any test has `output: {}` or is missing `result`/`valid`, fix it before submitting:
- For discovery runs (unknown expected price), use `"result": 0, "valid": true` as placeholders
- For excluded cases, use `"result": 0, "valid": false`

### Step 2 — Run tests

Call the `test_swallow_project` tool with the complete project JSON.

### Step 3 — Analyse results

For each test case:
- Show the test name, expected output, and actual output
- Flag any mismatches between expected and actual `result` or `valid`
- List any errors from the pricing engine (missing properties, expression failures, etc.)

### Step 4 — Debug failures

Read `result.debug.steps` to trace intermediate values. Find the FIRST step where values diverge — that's the bug.

For each failing test, trace the problem:

1. **All tests return identical results** — inputs are being ignored:
   - Check that every input field has both `key` and `exp` — without these, the engine ignores quote overrides and always uses `def`
   - Check that `exp` matches `key`: `key: "age"` → `exp: "{{age}}"`

2. **Result mismatch** — the price is wrong:
   - Check the step pipeline order — are properties available when referenced?
   - Check expression syntax — `{{property_name}}` wrapping, `==` not `===`, `!=` not `!==`
   - In calculation `formula` fields: use `and`/`or` not `&&`/`||` (MathJS syntax)
   - Check collection step syntax — starts with `collection.`, chains `.filter()`, data column first, never use `key` as a column name
   - Check `def` values — multiplicative factor defaults must be `1.0` not `0` (a `0` silently zeros the entire chain)
   - Check for unresolved `{{variable}}` in multiplication — silently becomes `0`, zeroing the chain
   - Check collection data is in compressed format: `[[headers], [row1], ...]`
   - `min()`/`max()` are NOT supported in transform `exp` fields — use calculation steps instead
   - Check `retain: true` on transform steps — without it, subsequent steps lose ALL prior values

3. **Valid mismatch** — exclusion/refer logic is wrong:
   - Check the `valid` expression in output — should be `{{exclusions.count()}} == 0`
   - Check exclusion step keys — never use `"exclusions"` as a key (shadows the built-in aggregate). Use prefixed names like `"uw_exclusions"`
   - Check exclusion item expressions — they trigger on `true`, so `{{age}} < 17` excludes ages under 17

4. **Engine errors** — missing properties, expression failures:
   - Check step ordering — properties must be defined before they're referenced
   - Check for typos in `{{property_name}}` references
   - Check that all input fields have `key` and `exp` matching their property name

### Step 5 — Summarise

Report:
- Total tests: N passed / N failed out of N
- Common failure patterns (if any)
- Specific fixes applied

ALL tests must pass — zero exceptions. A model with ANY failing test is BROKEN, not "close enough".

### Step 6 — Fix and re-test

If failures were found:
1. Apply the fixes to the project JSON
2. Call `validate_swallow_project` to check schema validity
3. Call `test_swallow_project` again to verify fixes
4. Repeat until all tests pass

Never set `expected.result` to match engine output to force a pass — fix the model instead. Never remove or weaken failing tests.

## Common issues

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| All results identical regardless of input | Missing `key`/`exp` on input fields | Add `key` and `exp` to every input format property |
| All results are 0 | Unresolved `{{variable}}` in multiplication chain | Check step order; ensure property is defined before use |
| All results are 0 | Multiplicative factor `def` is `0` | Change to `1.0` for factors, keep `0` only for additive components |
| All tests invalid | `valid.exp` references step key instead of aggregate | Use `{{exclusions.count()}} == 0` not `{{uw_exclusions.count()}} == 0` |
| No tests invalid | Exclusion step key is `"exclusions"` shadowing aggregate | Rename step key to `"uw_exclusions"` |
| Result slightly off | Rounding issue | Add `round(..., 2)` to result formula |
| Result NaN | Non-numeric input in calculation | Check input types are `integer` or `decimal` |
| Collection returns default | Filter syntax wrong | Ensure data column first: `filter(age=={{age}})` not `filter({{age}}==age)` |
| Downstream steps lose values | Missing `retain: true` on transform | Add `retain: true` to every transform step |
| `&&` or `||` fails in formula | MathJS doesn't support JS operators | Use `and`/`or` in calculation formulas; `&&`/`||` only in exclusion/refer `exp` |
