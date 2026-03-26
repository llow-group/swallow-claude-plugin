---
name: build-pricing-model
description: Build a Swallow pricing engine configuration from a natural language description of an insurance product
tools:
  - MCP
  - Read
  - Write
---

# Build Pricing Model

The user wants to create a Swallow pricing engine configuration. Follow this process:

1. **Understand the product** — Ask clarifying questions if needed: what's being insured, what factors affect the price, what exclusions exist, what the output fields are.

2. **Build the input schema** — Define all quote input fields. Every input MUST have `key`, `exp`, `label`, `sub_label` (min 20 chars, broker-facing), `type`, and `def`. Without `key` and `exp`, the engine ignores quote overrides and always uses `def`. Input types: `string`, `number`, `integer`, `decimal`, `boolean`, `date` only.

3. **Design the step pipeline** — Every step MUST have `id`, `step`, `name`, `key`, and `description`. Work through the logic in order:
   - Collection steps for rating table lookups (compressed format: `[[headers], [row1], ...]`, must have `retain: true`)
   - Transform steps to parse/normalise/combine values (must have `retain: true`; `min()`/`max()` NOT supported in `exp`)
   - Exclusion steps for validity rules
   - Refer steps for manual review triggers
   - Calculation steps for premium, commission, tax (MathJS: use `and`/`or` not `&&`/`||`)
   - Excess and endorsement steps if needed

   **Important:** Never use a step type name or its plural as a step key — e.g. never `key: "exclusion"`, `"exclusions"`, `"refer"`, `"refers"`. These collide with Swallow's built-in aggregate variables. Use descriptive prefixed names (e.g. `"uw_exclusions"`, `"risk_refers"`). Never use `key` as a data column name in collections. Never use `factors`, `modular`, `batch`, `links`, `label`, or `code` steps.

4. **Define the output** — Set the `result` formula, `valid` expression, and output format. Use `{{exclusions.count()}} == 0` in the `valid` expression. `output.format` is mandatory — expose per-coverage premiums and key intermediate values. Multiplicative factor defaults must be `1.0` not `0`.

5. **Add tests** — Create at least 3 test cases: happy path, exclusion trigger, and edge case. Every test must include an `output` object with `result` (number) and `valid` (boolean) — never leave `output` as `{}`. For excluded cases, set `valid: false` and `result: 0`. ALL tests must pass — zero exceptions.

6. **Validate** — Call `validate_swallow_project` via MCP to check the schema.

7. **Test** — Call `test_swallow_project` via MCP to run the pricing engine and verify results match expectations. If they don't, read `result.debug.steps` to trace intermediate values and find the FIRST step where values diverge.

8. **Iterate** — Fix any validation errors or test mismatches and re-run until all tests pass. Never set expected results to match engine output to force a pass — fix the model instead.

Write the final project JSON to a file for the user.
