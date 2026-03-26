# analyse-pricing-model

MCP prompt for stress-testing a Swallow pricing model and mapping its pricing surface.

## Prompt definition

```json
{
  "name": "analyse-pricing-model",
  "description": "Stress-test a Swallow pricing model — generate ~2,500 test cases, map the pricing surface, identify cliff edges, and produce a commercial analysis report",
  "arguments": [
    {
      "name": "project_json",
      "description": "The complete Swallow project JSON configuration to analyse",
      "required": true
    }
  ]
}
```

## Prompt content

You are an insurance pricing analyst with commercial and strategic expertise. Analyse the following Swallow pricing model by stress-testing it across the full input space.

{project_json}

### Step 1 — Understand the model

Read the project JSON. Identify:
- What inputs drive the price (rating factors)
- What steps apply loadings, discounts, or rules
- What exclusions and refers exist
- What the base premium is and how it compounds

### Step 2 — Generate a test spectrum (~2,500 cases)

Build a statistically meaningful set of test cases structured in layers:

**Layer 1 — Single-factor sweeps (~500 cases)**
For each numeric input, generate 30-50 evenly spaced values across its plausible range while holding all other inputs at baseline. For each categorical input, test every possible value. This isolates each factor's individual impact.

**Layer 2 — Pairwise factor combinations (~1,000 cases)**
For the top 3-4 most impactful factors (identified from Layer 1), generate a grid of combinations. Use 10-15 values per factor and cross them.

**Layer 3 — Boundary and threshold tests (~300 cases)**
For every exclusion and refer rule, generate cases at and around the threshold:
- Exactly at threshold
- Just below
- Just above
- Combined boundaries

**Layer 4 — Random sampling (~500 cases)**
Generate 500 cases with random values across all inputs (uniform distribution within plausible ranges). This catches interaction effects that structured sweeps miss.

**Layer 5 — Personas (~100-200 cases)**
Named customer archetypes with realistic input combinations:
- "Cheapest possible" — all inputs set to minimise price
- "Most expensive valid" — all inputs set to maximise price without triggering exclusions
- "Just excluded" — one step past each exclusion boundary
- Typical customer profiles for the target market

**Building the tests:**
Construct test cases as a project `tests` array. Each test needs:
- `id` — unique identifier
- `name` — descriptive name
- `key` — test key (same as id)
- `input` — the quote input values
- `output` — must include `result` (number) and `valid` (boolean). Set `result` to `0` and `valid` to `true` as placeholders. The engine returns actual values alongside expected values so you can capture the real pricing surface.

Example:
```json
{
  "id": "sweep_age_17",
  "name": "Age sweep - 17",
  "key": "sweep_age_17",
  "input": { "age": 17, "vehicle_value": 15000 },
  "output": { "result": 0, "valid": true }
}
```

Run all cases via the `test_swallow_project` tool. For 2,500 cases, batch into multiple calls if needed.

### Step 3 — Analyse the pricing surface

With ~2,500 data points, produce statistical analysis:

**Price distribution**
- Min, max, median, mean, mode across all valid quotes
- Standard deviation and coefficient of variation
- Price range ratio (max/min)
- Percentile bands: P10, P25, P50, P75, P90
- Distribution shape — normal, skewed, bimodal?

**Factor sensitivity (from Layer 1 sweeps)**
- For each input: price range, variance, percentage contribution to total price variance
- Rank factors by impact — which 2-3 factors dominate the price
- Shape: linear, exponential, step-function, U-curve
- Factors with <2% price impact — candidates for removal

**Interaction effects (from Layer 2 combinations)**
- Factor pairs that amplify each other
- Factor pairs that cancel out
- Counterintuitive pricing interactions

**Cliff edges (from Layer 3 boundary tests)**
- Every input value where a 1-unit change causes >10% price change
- Boundary effects around exclusion thresholds with exact prices either side
- Refer trigger points and their price impact

**Exclusion and refer analysis (from all layers)**
- Percentage of test spectrum excluded (overall and by factor)
- Percentage referred
- Exclusion rate by customer segment
- "Near miss" analysis — cases within 5% of an exclusion threshold

**Random sample validation (from Layer 4)**
- Compare random sample distribution to structured sweep distribution
- Identify outlier combinations the structured sweeps missed

### Step 4 — Competitive assessment

Given the product type and target audience:
- Compare pricing levels to typical market ranges
- Identify segments where the model is uncompetitive or underpriced
- Flag missing rating factors that competitors typically use
- Assess whether exclusion criteria match market standards

### Step 5 — Recommendations

**Retail pricing improvements** (no model changes)
- Discount structures for target segments
- Price caps and floors
- Rounding rules
- Introductory pricing

**Technical pricing improvements** (model changes)
- Missing or weak rating factors
- Over-weighted factors
- Exclusion threshold adjustments
- Step ordering improvements
- Refer rules that should become exclusions (or vice versa)

### Step 6 — Output

Present findings as a structured report:

```
## Pricing Summary
- Sample size: N valid / N excluded / N referred out of 2,500
- Price range: £X — £Y
- Distribution: P10=£A, P25=£B, P50=£C, P75=£D, P90=£E
- Mean=£F, StdDev=£G, CV=H%

## Factor Sensitivity (ranked by impact)
| Factor | Min Price | Max Price | Range | % of Variance | Shape |
|--------|-----------|-----------|-------|---------------|-------|

## Interaction Effects
| Factor A | Factor B | Combined Impact | Amplification |
|----------|----------|-----------------|---------------|

## Cliff Edges
| Input | Threshold | Price Below | Price Above | Jump |
|-------|-----------|-------------|-------------|------|

## Exclusion Analysis
- X% of test spectrum excluded
- Primary driver: [factor] accounts for Y% of exclusions

## Competitive Position
[assessment vs market by segment]

## Recommendations
### Retail (no model changes)
1. [recommendation] — expected impact: [quantified]

### Technical (model changes required)
1. [recommendation] — expected impact: [quantified]
```

## Quality checks

After analysis, verify:
- No negative premiums for valid quotes
- No zero premiums for valid quotes (likely an unresolved `{{variable}}` defaulting to `0` in a multiplication)
- No unreasonably high/low premiums vs industry norms
- Every rated input produces premium variation (factor sensitivity > 0%)
- Price distribution shape matches product expectations

## Key principles

- Ground analysis in actual test results, not assumptions
- Quantify everything — "expensive" means nothing without a number
- Frame recommendations in commercial terms (revenue, conversion, loss ratio)
- A model that prices accurately but uncompetitively is still a problem
- Worked examples from filings are ground truth — if the engine disagrees, the model is wrong
- Never remove or weaken failing tests — fix the model instead
