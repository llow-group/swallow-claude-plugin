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

2. **Build the input schema** — Define all quote input fields with appropriate types, labels, and defaults.

3. **Design the step pipeline** — Work through the logic in order:
   - Transform steps to parse/normalise inputs
   - Collection steps for rating table lookups
   - More transforms to combine factors
   - Exclusion steps for validity rules
   - Refer steps for manual review triggers
   - Calculation steps for premium, commission, tax
   - Excess and endorsement steps if needed

   **Important:** Never use a step type name or its plural as a step key — e.g. never `key: "exclusion"`, `"exclusions"`, `"refer"`, `"refers"`. These collide with Swallow's built-in aggregate variables. Use descriptive prefixed names instead (e.g. `"uw_exclusions"`, `"risk_refers"`).

4. **Define the output** — Set the `result` formula, `valid` expression, and any additional output fields. Use `!{{exclusions}}` in the `valid` expression to check the built-in exclusions aggregate.

5. **Add tests** — Create at least 3 test cases: happy path, exclusion trigger, and edge case. Every test must include an `output` object with `result` (number) and `valid` (boolean) — never leave `output` as `{}`. For excluded cases, set `valid: false` and `result: 0`.

6. **Validate** — Call `validate_swallow_project` via MCP to check the schema.

7. **Test** — Call `test_swallow_project` via MCP to run the pricing engine and verify results match expectations.

8. **Iterate** — Fix any validation errors or test mismatches and re-run until all tests pass.

Write the final project JSON to a file for the user.
