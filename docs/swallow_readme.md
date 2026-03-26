# Swallow Pricing Engine

A configurable pricing engine that takes a project definition (JSON config) and a quote payload and produces a price with associated decision rules (exclusions, excesses, endorsements, refers).

Used widely for fiscal products in Banking and Insurance.

## Installation

```bash
npm install @llow-group/swa_llow_pricing_engine
```

## Quick Start

```javascript
const { engine } = require('@llow-group/swa_llow_pricing_engine');

const project = {
    meta: {
        name: 'Simple Project',
        description: 'Simple motor premium'
    },
    input: {
        format: {
            base: {
                key: 'base',
                label: 'Base value',
                sub_label: 'The starting premium before any factors are applied.',
                def: 1000.00,
                type: 'decimal',
                exp: '{{base}}',
                static: true
            }
        }
    },
    steps: [{
        id: 'calc_premium',
        step: 'calculation',
        name: 'Premium',
        key: 'premium',
        def: 0,
        formula: '{{base}} * 10.91'
    }],
    output: {
        format: {},
        result: {
            key: 'result',
            formula: 'round({{premium}}, 2)',
            type: 'decimal',
            def: 0
        },
        valid: {
            key: 'output',
            exp: '{{exclusions.count()}} == 0',
            def: true,
            type: 'boolean'
        }
    }
};

const result = await engine({
    project,
    quote: { base: 500 },
});

// Output (always the same schema)
// {
//     result: 5455,
//     valid: true,
//     debug: {
//         errors: [],
//         exclusions: [],
//         refers: [],
//         excesses: [],
//         endorsements: [],
//         indicators: {},
//         meta: { ... },
//         timer: [ ... ],
//         steps: { ... }
//     }
// }
```

## How It Works

The engine processes data through a pipeline:

```
Quote Payload
    |
Input Block  (validate, map, type-check the incoming payload)
    |
Steps []     (transform, calculate, enrich, apply rules - in sequence)
    |
Output Block (format result, calculate final price, determine validity)
    |
Standardised Response
```

Properties flow through each step. The result of the input block is passed to the first step, and each step's result is passed to the next. All the way through to the output block.

When creating steps, reference previous results with template variables: `{{property_name}}`.

## Engine Parameters

```javascript
const result = await engine({
    project,           // Project config (JSON) - required
    quote,             // Quote payload (object) - required
    environment: {     // API config for external/modular steps - optional
        root_url: 'https://api.llow.io/sw/quotes',
        client_id: '',
        client_secret: '',
    },
    is_test: false,    // Skip input mapping when true - optional
    source: 'api',     // 'api', 'form', or 'test' - optional
    log: {},           // External log state - optional
    debug: true,       // Include debug object in response - optional (default true)
});
```

## Project Config

A project config has four top-level sections: `meta`, `input`, `steps`, and `output`.

### Meta

```javascript
{
    meta: {
        name: 'My Project',
        description: 'Description of the project',
        summary: 'Detailed model summary — what it rates, jurisdiction, methodology, audience.',
        project_reference: 'uuid',
        updated_at: '2024-01-01T00:00:00Z'
    }
}
```

The `summary` field should describe what the model rates, its jurisdiction, the rating methodology (e.g. multiplicative factor chain, scored dimensions), and target audience.

### Input Block

The input block sets up the data contract with the quote payload. It validates types and maps incoming data to internal property names.

```javascript
{
    input: {
        // Optional - transform payload before format processing
        mapping: `
            let result = {
                name: proposer.name,
                age: proposer.age,
                total_claims: drivers.map(d => d.claims).reduce((sum, c) => sum + c, 0)
            };
            return result;
        `,
        // Optional - set true when payload is an array
        batch: false,
        // Optional - child format definitions for array-type inputs
        format_children: {
            driver_format: {
                ncd: { key: 'ncd', label: 'NCD', exp: '{{ncd}}', type: 'integer', def: 0 },
                dob: { key: 'dob', label: 'DOB', exp: '{{dob}}', type: 'date', def: '2000-01-01' }
            }
        },
        // Required - defines the input properties
        format: {
            proposer_name: {
                key: 'proposer_name',
                label: 'Name',
                sub_label: 'Full name of the proposer as it appears on the policy.',
                exp: '{{proposer.name}}',
                type: 'string',
                def: 'Unknown',
                index: true
            },
            proposer_age: {
                key: 'proposer_age',
                label: 'Age',
                sub_label: 'Proposer age in years, derived from date of birth.',
                exp: '{{proposer.dob.age(YY)}}',
                type: 'integer',
                def: 18
            },
            additional_drivers: {
                key: 'additional_drivers',
                label: 'Drivers',
                sub_label: 'Array of additional driver objects with NCD and date of birth.',
                exp: '{{additional_drivers}}',
                type: 'array',
                def: [],
                format_children: 'driver_format'
            },
            base_price: {
                key: 'base_price',
                label: 'Base price',
                sub_label: 'Internal base premium before any rating factors. Not user-editable.',
                type: 'decimal',
                def: 1000.00,
                static: true
            }
        }
    }
}
```

**Format property fields:**

| Field | Required | Description |
|-------|----------|-------------|
| `key` | Yes | The property key. Must match the format object key. **Without this, quote overrides are silently ignored.** |
| `label` | Yes | Human-readable label (think: form field header) |
| `sub_label` | Recommended | Detailed explanation — what to enter, valid values, where to find the value |
| `type` | Yes | Data type: `string`, `integer`, `decimal`, `number`, `currency`, `boolean`, `date`, `array`, `object`, `enum` |
| `exp` | Yes | Expression to extract value from payload. Uses `{{path}}` syntax. Typically `{{key_name}}` |
| `def` | Yes | Default value when expression fails or input is missing |
| `items` | No | Array of valid values — used for UI dropdowns and test generation |
| `index` | No | If `true`, value is stored as a searchable indicator (max 4) |
| `static` | No | If `true`, value is internal and cannot be overwritten by quote input |
| `format_children` | No | Reference to a child format definition (for array types) |

> **Critical:** Every input MUST have both `key` and `exp` properties. Without them, the engine ignores quote overrides and always uses `def` — meaning all tests return identical results regardless of input. This is the single most common cause of "all tests return the same value" bugs.

**Mapping block** (optional): A JavaScript string that pre-processes the raw payload before the format block runs. Must return an object or array. Only runs when `source` is `'api'` (not `'form'`).

**Batch mode** (optional): Set `batch: true` when the incoming payload is an array. Each item in the array is processed through the input format independently.

### Steps

Steps are an ordered array of processors. Each step receives the accumulated result of all previous steps.

```javascript
{
    steps: [
        {
            id: 'unique_id',
            step: 'transform',       // Step type
            name: 'Step Name',       // Display name
            description: 'What this step does and why it matters.',
            key: 'step_key',         // Result property name
            placeholder: false,      // If true, skip execution and retain data
            retain: true,            // If true, merge output with previous data
            // ... step-specific config
        }
    ]
}
```

All steps support `placeholder: true` which skips execution and passes data through unchanged. This is useful for disabling steps without removing them.

**Step `description`** — Every step should have a `description` explaining:
- **collection steps**: What factor/rate it looks up, dimensions, how values affect premium. E.g. *"Looks up protection class factor by state and class (85 rows, classes 1-10). Higher classes increase premium — 1.25x for class 10 vs 0.35x for class 1."*
- **transform steps**: What derived values it computes and why. E.g. *"Computes adjusted Coverage A limit (limit x RC adjustment) needed for Key Factor lookup."*
- **calculation steps**: Formula pattern and whether it produces a final or intermediate value. E.g. *"Multiplies 14 rating factors, rounds to nearest dollar. Core premium before endorsements."*

#### Step Types

---

#### `transform`

Maps and transforms properties into new values.

```javascript
{
    id: 'age_transform',
    step: 'transform',
    name: 'Age Transform',
    key: 'age_transform',
    retain: true,
    format: {
        age_months: {
            key: 'age_months',
            label: 'Age in months',
            exp: '{{age_years}} * 12',
            type: 'integer',
            def: 0
        },
        postcode_area: {
            key: 'postcode_area',
            label: 'Postcode area',
            exp: '{{postcode.regex(^[A-Za-z]{1,2})}}',
            type: 'string',
            def: 'X'
        }
    }
}
```

When `retain: true`, the output is merged with the input so all previous properties remain available to subsequent steps.

> **Critical:** Every transform step MUST have `retain: true`. Without it, subsequent steps lose access to ALL prior values and silently fall back to defaults. This is a common and devastating bug — the model appears to work but produces wrong results because downstream steps can't see upstream outputs.

> **Limitation:** `min()` and `max()` functions are NOT supported in transform `exp` fields. Use individual `calculation` steps instead. `round()` works in both.

---

#### `calculation`

Evaluates a mathematical formula. Uses [mathjs](https://mathjs.org/) for evaluation.

```javascript
{
    id: 'calc_premium',
    step: 'calculation',
    name: 'Premium',
    key: 'premium',
    def: 0,
    formula: 'round({{base_price}} * (25 / {{age}}) * {{factor}}, 2)'
}
```

Calculations always retain previous data. If the formula fails (e.g. division by zero, non-numeric input), the `def` value is used and an error is logged.

**Common formula patterns:**

```javascript
// Multiplicative factor chain (most common in insurance)
formula: 'round({{base_rate}} * {{territory_factor}} * {{age_factor}} * {{class_factor}}, 2)'

// Per-step rounding (some filings round after each factor)
formula: 'round(round({{base_rate}} * {{factor_1}}, 2) * {{factor_2}}, 2)'

// Final-only rounding (other filings round once at the end)
formula: 'round({{base}} * {{f1}} * {{f2}} * {{f3}} * {{f4}}, 0)'

// Ternary conditional
formula: '{{is_good_driver}} == true ? {{premium}} * 0.80 : {{premium}}'

// Additive with conditional components
formula: '{{base_premium}} + {{endorsement_premium}} + {{policy_fee}}'

// Minimum premium floor
formula: 'max({{calculated_premium}}, {{minimum_premium}})'
```

---

#### `factors`

Rating factor tables that produce weights, exclusions, refers, excesses, and endorsements. Factors map input dimensions to lookup tables (hashes).

```javascript
{
    id: 'driving_factors',
    step: 'factors',
    key: 'driving_factors',
    result: {
        key: 'driving_factors',
        formula: 'round({{age_factor}} * {{ncd_factor}}, 4)',
        type: 'decimal',
        def: 0
    },
    factors: {
        age_factor: {
            def: { weight: 1, exclude: false, refer: false, excess: 0, endorsement: '' },
            dimensions: {
                age: {
                    value: ['18', '25', '35', '50'],
                    value_to: ['24', '34', '49', '65']  // Optional range upper bounds
                }
            },
            hashes: {
                '18': { weight: 1.8, exclude: false, refer: true, excess: 500, endorsement: 'Y1' },
                '25': { weight: 1.2, exclude: false, refer: false, excess: 0, endorsement: '' },
                '35': { weight: 1.0, exclude: false, refer: false, excess: 0, endorsement: '' },
                '50': { weight: 0.9, exclude: false, refer: false, excess: 0, endorsement: '' }
            }
        }
    }
}
```

Multi-dimensional factors use `::` as a separator in hash keys (e.g. `'W::1'` for postcode `W` and children `1`).

Each factor hash can produce:
- `weight` - numeric multiplier used in the result formula
- `exclude` - if `true`, adds an exclusion
- `refer` - if `true`, adds a referral
- `excess` - if non-zero, adds an excess
- `endorsement` - if non-empty, adds an endorsement

---

#### `collection`

Looks up values from a data collection (array of objects or compressed format) using filter/map query syntax.

```javascript
{
    id: 'postcode_lookup',
    step: 'collection',
    name: 'Postcode Lookup',
    key: 'postcode_lookup',
    collection: [
        { area: 'W', amount: 500, rate: 1.2 },
        { area: 'E', amount: 300, rate: 1.0 },
        { area: 'N', amount: 400, rate: 1.1 }
    ],
    format: {
        postcode_amount: {
            key: 'postcode_amount',
            label: 'Postcode amount',
            exp: 'collection.filter(area={{postcode_area}}).map(amount).first()',
            type: 'integer',
            def: 0
        }
    }
}
```

**Compressed format** — for large tables (100+ rows), use compressed arrays to reduce file size. The engine accepts both formats natively:

```javascript
{
    id: 'territory_rates',
    step: 'collection',
    name: 'Territory Rate Lookup',
    key: 'territory_rates',
    description: 'Territory base rate by ZIP code. 2,400 rows covering all rated territories.',
    collection: [
        ["zip", "territory", "bi_rate", "pd_rate", "comp_rate"],
        ["90001", "LA", 247.00, 189.50, 95.20],
        ["90002", "LA", 247.00, 189.50, 95.20],
        ["10001", "NYC", 312.00, 245.80, 128.40]
        // ... all rows
    ],
    format: {
        bi_base_rate: {
            key: 'bi_base_rate',
            label: 'BI Base Rate',
            exp: 'collection.filter(zip={{zip_code}}).map(bi_rate).first()',
            type: 'decimal',
            def: 0
        }
    }
}
```

> **Reserved word:** Do NOT use `key` as a column name in collection data — it is reserved by the engine. Use alternatives like `code`, `band`, `lookup_key`, `score_key`.

**Filter patterns:**

```javascript
// Single-key exact match (most common)
'collection.filter(zip={{zip_code}}).map(rate).first()'

// Multi-key chained filters
'collection.filter(state={{state}}).filter(class={{class_code}}).map(factor).first()'

// Range filter (for banded lookups like age bands, mileage tiers)
'collection.filter(min<={{annual_mileage}}).filter(max>={{annual_mileage}}).map(factor).first()'

// Aggregation
'collection.map(result).sum()'
'collection.sum(premium)'
'collection.count()'

// First/last from filtered set
'collection.filter(tier={{risk_tier}}).map(rate).first()'
'collection.filter(tier={{risk_tier}}).map(rate).last()'
```

> **Empty result behavior:** When a filter matches no rows, the `def` value is used. In a multiplicative chain, a default of 0 will zero the entire premium. Set multiplicative factor defaults to 1.0, not 0.

---

#### `links`

Maps a single identifier to a set of enrichment values. Like a dictionary/lookup table.

```javascript
{
    id: 'vehicle_enrichment',
    step: 'links',
    key: 'vehicle_enrichment',
    exp: '{{vehicle_body_type}}',
    format: {
        vehicle_weight: { key: 'vehicle_weight', label: 'Weight', def: 1000, type: 'integer' },
        vehicle_rating: { key: 'vehicle_rating', label: 'Rating', def: 1, type: 'integer' }
    },
    links: {
        '2DH': { vehicle_weight: 1000, vehicle_rating: 1 },
        '4DH': { vehicle_weight: 1400, vehicle_rating: 4 },
        'MPV': { vehicle_weight: 2000, vehicle_rating: 5 }
    }
}
```

The `exp` is evaluated to get a key, which is then looked up in the `links` map. If not found, format defaults are used.

---

#### `label`

Creates a category/label from conditional rules. The last matching item wins.

```javascript
{
    id: 'age_band',
    step: 'label',
    key: 'age_band',
    def: 'unknown',
    items: [
        { name: 'Young', exp: '{{age}} < 25', value: 'young' },
        { name: 'Mid', exp: '({{age}} >= 25 and {{age}} < 50)', value: 'mid' },
        { name: 'Senior', exp: '{{age}} >= 50', value: 'senior' }
    ]
}
```

---

#### `exclusion`

Boolean tests that exclude (reject) a quote. If any item evaluates to `true`, the exclusion is triggered and the quote becomes invalid.

```javascript
{
    id: 'my_exclusions',
    step: 'exclusion',
    key: 'my_exclusions',
    items: [
        { id: 'too_young', name: 'Too young', exp: '{{age}} < 17', def: false },
        { id: 'banned', name: 'Banned driver', exp: '{{is_banned}} == true', def: false },
        {
            id: 'gap_requires_comp_coll',
            name: 'GAP requires COMP and COLL',
            exp: '({{has_gap}} == true) and ({{comp_limit}} == "no_coverage" or {{coll_limit}} == "no_coverage")',
            def: false
        }
    ]
}
```

Triggered exclusions are collected in the output's `exclusions` array. When any exclusion triggers, `output.valid` becomes `false`.

---

#### `refer`

Same structure as exclusions but for flagging quotes for manual review rather than rejecting them.

```javascript
{
    id: 'my_refers',
    step: 'refer',
    key: 'my_refers',
    items: [
        { id: 'expensive_vehicle', name: 'Expensive vehicle', exp: '{{vehicle_value}} > 75000', def: false }
    ]
}
```

Triggered refers are collected in the output's `refers` array. They do NOT affect `output.valid`.

---

#### `endorsements`

Conditional string labels added to a quote. Each matching item appends its `value` to the endorsements list.

```javascript
{
    id: 'my_endorsements',
    step: 'endorsements',
    key: 'my_endorsements',
    def: [],
    items: [
        { id: 'end_good_driver', name: 'Good Driver Discount', exp: '{{is_good_driver}} == true', value: 'GOOD_DRIVER' },
        { id: 'end_multi_policy', name: 'Multi-Policy', exp: '{{is_multi_policy}} == true', value: 'MULTI_POLICY' }
    ]
}
```

Duplicate endorsement values are automatically removed (unique only).

---

#### `excesses`

Conditional numeric values added as compulsory excess amounts.

```javascript
{
    id: 'my_excesses',
    step: 'excesses',
    key: 'my_excesses',
    def: [200.00],
    items: [
        { id: 'young_driver', name: 'Young driver', exp: '{{age}} < 25', value: 100.00 },
        { id: 'expensive_car', name: 'Expensive car', exp: '{{vehicle_value}} > 100000', value: 400.00 }
    ]
}
```

---

#### `external`

Calls an external API and maps the response into properties.

```javascript
{
    id: 'credit_check',
    step: 'external',
    key: 'credit_check',
    options: {
        url: 'https://api.example.com/credit',
        method: 'post',
        timeout: 5000,
        headers: { auth: 'secret_key' },
        data: { id: 150 },
        mock: false  // Set true in tests to skip the API call and use format defaults
    },
    format: {
        credit_score: {
            key: 'credit_score',
            label: 'Credit score',
            exp: '{{json.score}}',
            type: 'integer',
            def: 0
        }
    }
}
```

For testing, set `options.mock: true` to bypass the API and return format defaults. You can also use `mock::<format_key>` in the quote input to inject specific mock values per field.

---

#### `code`

Executes custom JavaScript code. The code has access to all current properties and must return an object.

```javascript
{
    id: 'custom_check',
    step: 'code',
    key: 'custom_check',
    code: `
        if (premium > 100) {
            return { is_high: true };
        } else {
            return { is_high: false };
        }
    `,
    freeform: false,
    format: {
        is_high: {
            key: 'is_high',
            label: 'Is high premium',
            exp: '{{is_high}}',
            type: 'boolean',
            def: false
        }
    }
}
```

The code string runs in a sandboxed context with access to lodash (`_`) and mathjs functions. When `freeform: true`, the code receives the raw object; otherwise property names are sanitised.

> **Warning:** `code` steps are NOT schema-compliant in auto-generated models and will fail the schema validation gate. For array iteration, use `modules` steps instead (see below).

---

#### `batch`

Sends an array of quotes to another Swallow project via API and collects the results as a collection.

```javascript
{
    id: 'batch_quotes',
    step: 'batch',
    key: 'batch_quotes',
    options: {
        project_reference: 'uuid',
        version_reference: 'uuid'
    },
    format: {
        total_premium: {
            key: 'total_premium',
            label: 'Total',
            exp: 'collection.map(result).sum()',
            type: 'decimal',
            def: 0
        }
    }
}
```

Quotes are chunked into batches of 100 and processed in parallel.

---

#### `modular`

Calls another Swallow project via API and maps the response.

```javascript
{
    id: 'sub_project',
    step: 'modular',
    key: 'sub_project',
    options: {
        project_reference: 'uuid',
        version_reference: 'uuid',
        timeout: 5000
    },
    format: {
        sub_result: {
            key: 'sub_result',
            label: 'Sub-project result',
            exp: '{{result}}',
            type: 'decimal',
            def: 0
        }
    }
}
```

Set `timeout: 0` to skip the API call and return format defaults.

---

#### `modules`

Runs an embedded project (provided in the project's `modules` array) against the current data. Like `modular` but without an API call — everything is self-contained.

```javascript
{
    id: 'embedded_module',
    step: 'modules',
    key: 'embedded_module',
    options: {
        project_reference: 'uuid',
        version_reference: 'uuid',
        root_property: '',   // Optional - send a sub-property instead of entire object
        debug: false         // Optional - include debug info from module run
    },
    format: {
        module_result: {
            key: 'module_result',
            label: 'Module result',
            exp: 'collection.map(result).first()',
            type: 'decimal',
            def: 0
        }
    }
}
```

##### Multi-Item Rating with Modules

Some products require rating **multiple items per policy** — multiple travelers (travel insurance), multiple vehicles (fleet auto), multiple locations (farm/commercial property), multiple class codes (workers comp). The engine cannot loop over variable-length arrays in standard collection/calculation steps.

**Use the module step pattern** with `root_property` pointing to an array input. The engine iterates over each element, runs the child model for each, then aggregates.

**Parent config:**

```javascript
{
    input: {
        format: {
            travelers: {
                key: 'travelers',
                label: 'Travelers',
                sub_label: 'Array of traveler objects with age and coverage selections.',
                type: 'array',
                def: [],
                exp: '{{travelers}}'
            }
        }
    },
    steps: [
        {
            id: 'traveler_rating',
            step: 'modules',
            name: 'Per-Traveler Rating',
            key: 'traveler_rating',
            options: {
                root_property: 'travelers',
                project_reference: 'uuid-1',
                version_reference: 'uuid-2',
                debug: false
            },
            format: {
                total_premium: {
                    key: 'total_premium',
                    label: 'Total Premium',
                    exp: 'collection.sum(result)',
                    type: 'decimal',
                    def: 0
                },
                tcan_total: {
                    key: 'tcan_total',
                    label: 'Trip Cancellation Total',
                    exp: 'collection.sum(tcan_premium)',
                    type: 'decimal',
                    def: 0
                }
            }
        }
    ],
    modules: [
        {
            root_property: 'travelers',
            project_reference: 'uuid-1',
            version_reference: 'uuid-2',
            project: {
                // Complete child Swallow config
                input: {
                    format: {
                        age: { key: 'age', label: 'Age', type: 'integer', def: 35, exp: '{{age}}' },
                        trip_cost: { key: 'trip_cost', label: 'Trip Cost', type: 'decimal', def: 1000, exp: '{{trip_cost}}' }
                    }
                },
                steps: [
                    { id: 'base_rates', step: 'collection', key: 'base_rates', /* ... */ },
                    { id: 'premium', step: 'calculation', key: 'premium', /* ... */ }
                ],
                output: {
                    result: { key: 'result', formula: '{{premium}}', type: 'decimal', def: 0 },
                    format: {
                        tcan_premium: { key: 'tcan_premium', exp: '{{tcan_premium}}', type: 'decimal', def: 0 }
                    },
                    valid: { key: 'output', exp: 'true', def: true, type: 'boolean' }
                }
            }
        }
    ]
}
```

**Quote payload:**

```javascript
{
    travelers: [
        { age: 35, trip_cost: 1500 },
        { age: 68, trip_cost: 1500 }
    ]
}
```

**When to use this pattern:**
- Travel insurance — travelers array (age-banded base rates per traveler)
- Fleet auto — vehicles array (per-vehicle class/territory/factor chain)
- Farm/commercial property — locations array (per-location coverage rating)
- Workers comp — class codes array (per-class payroll x rate)

### Output Block

The output block calculates the final price, determines validity, and formats additional output properties.

```javascript
{
    output: {
        // Optional - additional properties to include in the response
        format: {
            bi_premium: {
                key: 'bi_premium',
                label: 'BI Premium',
                sub_label: 'Bodily Injury liability premium after all factors.',
                exp: '{{bi_premium}}',
                type: 'decimal',
                def: 0
            },
            pd_premium: {
                key: 'pd_premium',
                label: 'PD Premium',
                sub_label: 'Property Damage liability premium after all factors.',
                exp: '{{pd_premium}}',
                type: 'decimal',
                def: 0
            },
            total_modifier: {
                key: 'total_modifier',
                label: 'Total Rating Modifier',
                sub_label: 'Combined experience and schedule rating modifier.',
                exp: '{{total_modifier}}',
                type: 'decimal',
                def: 1
            }
        },
        // Required - calculates the final price
        result: {
            key: 'result',
            formula: 'round({{premium}} + {{tax}} + {{commission}}, 2)',
            type: 'decimal',
            def: 0
        },
        // Required - determines if the quote is valid
        valid: {
            key: 'output',
            exp: '{{exclusions.count()}} == 0',
            def: true,
            type: 'boolean'
        },
        // Optional - JavaScript code to post-process the output
        mapping: `
            return {
                ...result,
                formatted_price: '£' + result.toFixed(2)
            };
        `
    }
}
```

The `valid` expression has access to `{{exclusions}}`, `{{refers}}`, `{{endorsements}}`, and `{{excesses}}` arrays collected during step processing.

**Output format fields are important** — they expose intermediate premiums as named output fields for debugging, spreadsheet cross-referencing, and downstream consumers. Every `output.format` entry should have: `key`, `label`, `sub_label`, `exp`, `type`, `def`.

## Expressions and Query Language

Expressions use `{{path}}` syntax powered by [Swallow QL](https://github.com/llow-group/swa_llow_ql).

### Expression Syntax Rules

The engine uses **MathJS** for calculation formulas and **Swallow QL** for property access. This is NOT JavaScript — different operators apply:

| Use this | NOT this | Context |
|----------|----------|---------|
| `==` | `===` | Equality |
| `!=` | `!==` | Inequality |
| `and` | `&&` | Logical AND (in MathJS formulas) |
| `or` | `\|\|` | Logical OR (in MathJS formulas) |
| `{{variable}}` | bare `variable` | Variable references |

> **Using `===`, `&&`, or `||` in calculation formulas will cause silent failures or wrong results.** The engine will not error — it will simply evaluate incorrectly.

> **Note:** In `exp` fields (non-formula contexts like exclusion/refer/label items), `&&` and `||` ARE supported as they use JavaScript evaluation. The MathJS restriction applies specifically to `formula` fields in calculation steps and format `exp` fields.

### Property Access

```
{{age}}                          // top-level property
{{proposer.name}}                // nested property
{{proposer.dob.age(YY)}}        // age in years from current date
{{proposer.dob.age(MM)}}        // age in months
{{proposer.dob.age(YY, start_date)}} // age relative to another date field
```

### Array Operations

```
{{drivers.count()}}              // array length
{{drivers.sum(claims)}}          // sum a property
{{drivers.map(ncd).min()}}       // minimum value
{{drivers.map(ncd).max()}}       // maximum value
{{drivers.map(ncd).mean()}}      // average value
{{drivers.map(ncd).range()}}     // max - min
{{drivers.filter(employment=E).count()}}  // filter then count
{{drivers.unique(code).count()}} // count unique values
{{drivers.map(code).first()}}    // first element
{{drivers.map(code).last()}}     // last element
```

### String Operations

```
{{postcode}}.regex(^[A-Za-z]{1,2})     // regex match
{{postcode}}.postcode(area)            // postcode area extraction
{{postcode}}.postcode(sector)          // postcode sector extraction
({{name}}).includes("Bob")             // string includes
({{postcode}}).startsWith("W")         // starts with
```

### Boolean Expressions

```
({{age}} > 25 and {{ncd}} >= 2)        // AND (in MathJS formula context)
({{age}} < 18 or {{is_banned}})        // OR (in MathJS formula context)
({{age}} > 25 && {{ncd}} >= 2)         // AND (in exp/exclusion context)
({{age}} < 18 || {{is_banned}})        // OR (in exp/exclusion context)
```

### Ternary Conditionals

```javascript
// Boolean check
'{{is_good_driver}} == true ? 0.80 : 1.00'

// String comparison (use double quotes inside)
"{{driver_class}} == 'good' ? {{good_factor}} : ({{driver_class}} == 'elite' ? {{elite_factor}} : 1.00)"

// Numeric threshold
'({{driving_record_points}} >= 5) ? {{surcharge}} : 1.00'

// Presence check
"({{cdw_limit}} != 'no_coverage') ? 1.05 : 1.00"
```

### Aggregation Helpers

```
min([{{drivers.map(age).min()}}, {{proposer_age}}])    // min of values
max([{{val_a}}, {{val_b}}])                            // max of values
```

### Date Operations

```
{{dob.date(YY)}}                 // extract year
{{dob.date(MM)}}                 // extract month
{{dob.date(DD)}}                 // extract day
{{dob.date(UK)}}                 // format UK date (DD/MM/YYYY → YYYY-MM-DD)
{{dob.date(US)}}                 // format US date (MM/DD/YYYY → YYYY-MM-DD)
{{dob.age(YY)}}                  // years since now
{{dob.age(MM)}}                  // months since now
{{dob.age(YY, inception_date)}}  // years since another field
```

### Default Values

```
{{country.default(GB)}}          // "GB" if country is undefined
{{height.default(180)}}          // 180 if height is undefined
```

### Existence Check

```
{{spouse.exists()}}              // "false" if null, undefined, empty string, or 0
{{title.exists()}}               // "true" if has a value
```

### Collection Query Syntax

Inside collection step `format.*.exp` fields:

```
collection.filter(column={{input_ref}}).map(value_col).first()
collection.filter(col1={{ref1}}).filter(col2={{ref2}}).map(value).first()
collection.filter(min<={{value}}).filter(max>={{value}}).map(factor).first()
collection.map(result).sum()
collection.sum(premium)
collection.count()
```

### Excel Formulas

Excel formulas are also supported as expressions:

```
SUM({{array}})
VLOOKUP({{key}}, {{table}}, 2, false)
IF({{age}} > 25, "adult", "young")
```

## Unresolved References

When a `{{variable}}` cannot be resolved (the property doesn't exist in the current state), the behaviour depends on context:

| Context | Behaviour | Danger |
|---------|-----------|--------|
| Calculation formula (multiplication) | Defaults to 0 | **Zeros the entire chain** |
| Calculation formula (addition) | Defaults to 0 | Silently drops the term |
| Collection filter | No rows match | Falls back to `def` value |
| Transform exp | Expression fails | Falls back to `def` value |

> **The most dangerous case:** An unresolved `{{variable}}` in a multiplicative formula silently becomes 0, which zeros the entire product: `{{base}} * {{missing_factor}} * {{other_factor}}` → `100 * 0 * 1.2` → `0`. Every variable reference MUST resolve to either an `input.format` key or a preceding step's output key.

## Default Value Strategy

Choose defaults carefully based on how the value is used:

| Usage | Safe default | Why |
|-------|-------------|-----|
| Multiplicative factor | `1.0` | Neutral multiplier — doesn't change the result |
| Additive component | `0` | Neutral addend — doesn't change the result |
| Rate/base premium | `0` | Makes the missing rate obvious (zero premium) |
| Boolean flag | `false` | Conservative — doesn't trigger rules |
| String enum | First/most common value | Produces a valid lookup result |

> **Never use 0 as the default for a multiplicative factor** — it will zero out the entire premium chain silently.

## Embedded Tests

Projects can include test cases that verify the model produces correct results:

```javascript
{
    tests: [
        {
            id: 'source_test_1',
            key: 'example_1',
            name: 'Filing Example 1 — ZIP 90036, 30/60 BI',
            category: 'source',     // From filing worked example
            input: {
                zip_code: '90036',
                bi_limit: '30_60',
                driver_age: 35,
                good_driver: true
            },
            output: {
                result: 3471.25,    // Must match filing exactly (penny-level)
                valid: true
            },
            mocks: {},              // Mock values for external steps
            tags: ['NB']
        },
        {
            id: 'exclusion_test',
            key: 'gap_without_comp',
            name: 'GAP without COMP should be excluded',
            category: 'synthetic',  // Rule verification
            input: {
                has_gap: true,
                comp_limit: 'no_coverage'
            },
            output: {
                result: null,
                valid: false
            }
        }
    ]
}
```

**Test categories:**
- `category: "source"` — inputs and expected premium taken directly from the filing's worked example. Must match penny-perfectly. At least one source test is required for a model to be considered valid.
- `category: "synthetic"` — scenario tests for exclusions, refers, edge cases. Validates internal consistency and business rules.

**Run embedded tests programmatically:**

```javascript
const { evaluateTests } = require('@llow-group/swa_llow_pricing_engine');

const results = await evaluateTests({ project });
// Returns pass/fail for each test case with full debug objects
```

## Debug Output

Every engine response includes a `debug` object (unless `debug: false` is passed):

```javascript
{
    result: 3471.25,
    valid: true,
    debug: {
        meta: { name: 'Model Name', /* ... */ },
        timer: [{ step: 'territory_lookup', ms: 2 }, /* ... */],
        errors: [],          // Validation/expression errors
        exclusions: [],      // Triggered exclusion rules
        refers: [],          // Triggered referral flags
        excesses: [],        // Applied excess amounts
        endorsements: [],    // Applied endorsement codes
        indicators: {        // Indexed input values
            zip_code: '90036'
        },
        steps: {
            // Every step's output value — trace the full factor chain
            territory_factor: 1.15,
            age_factor: 0.95,
            class_factor: 1.20,
            base_premium: 247.00,
            adjusted_premium: 340.76,
            final_premium: 3471.25
        }
    }
}
```

**Debugging strategy:** When a test fails, compare `debug.steps` against the expected intermediate values from the filing's worked example. Find the FIRST step where the value diverges — that's where the bug is. Common causes:
- Collection filter returning empty result (check column names match)
- Unresolved `{{variable}}` reference defaulting to 0
- Wrong operator (`===` instead of `==`)
- Missing `retain: true` on a transform (downstream step can't see the value)

## Common Gotchas

Hard-won lessons from building 50+ models. These are the issues that cause the most debugging time.

### 1. Missing `key` and `exp` on Input Format (Most Common)

```javascript
// WRONG — engine ignores quote overrides, always uses def
format: {
    zip_code: {
        label: 'ZIP Code',
        type: 'string',
        def: '90036'
    }
}

// CORRECT
format: {
    zip_code: {
        key: 'zip_code',
        exp: '{{zip_code}}',
        label: 'ZIP Code',
        type: 'string',
        def: '90036'
    }
}
```

**Symptom:** All tests return identical results regardless of input values.

### 2. Missing `retain: true` on Transform Steps

```javascript
// WRONG — downstream steps lose access to derived_limit
{ step: 'transform', key: 'prep', format: { derived_limit: { exp: '{{limit}} / 100' } } }

// CORRECT
{ step: 'transform', key: 'prep', retain: true, format: { derived_limit: { exp: '{{limit}} / 100' } } }
```

**Symptom:** Steps after the transform produce wrong results or fall back to defaults.

### 3. Factor Tables with All-1.0 Placeholder Values

```javascript
// WRONG — this is a placeholder, not real data
collection: [
    ["territory", "factor"],
    ["01", 1.0],
    ["02", 1.0],
    ["03", 1.0]
]
```

**Symptom:** Changing the input (e.g. territory from "01" to "03") produces no premium change. Two tests differing only in one factor input return identical results.

### 4. Using `===`/`&&`/`||` in Calculation Formulas

```javascript
// WRONG — MathJS doesn't support these operators
formula: '{{age}} === 25 ? 1.0 : ({{age}} > 25 && {{age}} < 65 ? 0.95 : 1.20)'

// CORRECT
formula: '{{age}} == 25 ? 1.0 : ({{age}} > 25 and {{age}} < 65 ? 0.95 : 1.20)'
```

**Symptom:** Formula silently evaluates incorrectly or returns `def`.

### 5. Using `key` as a Column Name in Collections

```javascript
// WRONG — 'key' is reserved by the engine
collection: [
    { key: 'W', rate: 1.2 },
    { key: 'E', rate: 1.0 }
]

// CORRECT — use a descriptive name
collection: [
    { area: 'W', rate: 1.2 },
    { area: 'E', rate: 1.0 }
]
```

**Symptom:** Collection lookups return undefined or wrong results.

### 6. Unresolved Variable References in Multiplication

```javascript
// DANGEROUS — if territory_factor doesn't exist, entire result is 0
formula: '{{base_rate}} * {{territory_factor}} * {{age_factor}}'
```

**Symptom:** Premium is $0 despite valid inputs. Check `debug.steps` — the unresolved variable shows as 0.

### 7. Wrong Default Values for Multiplicative Factors

```javascript
// WRONG — def: 0 will zero the chain if lookup fails
{ exp: 'collection.filter(zip={{zip}}).map(factor).first()', def: 0 }

// CORRECT — def: 1 is neutral in multiplication
{ exp: 'collection.filter(zip={{zip}}).map(factor).first()', def: 1 }
```

### 8. Empty Collection Data

```javascript
// WRONG — collection has no data rows
{ step: 'collection', collection: [], format: { /* ... */ } }
```

**Symptom:** All lookups return `def` values. The collection gate detects this.

### 9. Using `min()`/`max()` in Transform Expressions

```javascript
// WRONG — not supported in transform exp fields
{ step: 'transform', format: { capped: { exp: 'min({{value}}, 1000)' } } }

// CORRECT — use a separate calculation step
{ step: 'calculation', key: 'capped', formula: 'min({{value}}, 1000)', def: 0 }
```

### 10. Hardcoded Test Answers in Formulas

```javascript
// WRONG — embedding the expected test answer
formula: '({{zip_code}} == "90036") ? 3471.25 : {{calculated_premium}}'

// CORRECT — formulas must compute from factors and rate tables
formula: 'round({{base_rate}} * {{territory_factor}} * {{age_factor}}, 2)'
```

**Symptom:** Tests pass but the model produces wrong results for any input not in the test suite.

### 11. Gemini/LLM API Timeouts During Extraction

Gemini 3.1 preview models can timeout (300+ second hangs) with `fetch failed` errors. The pipeline handles this with automatic fallback to Gemini 2.5 Pro on retry. If building manually, switch models after 2 consecutive timeouts.

### 12. Insufficient Cohort Test Diversity

Cohort tests need 30+ valid test sets. Common causes of insufficient diversity:
- Input `items` arrays too restrictive (only 2-3 values)
- Collection data covers only a narrow range
- Too many exclusion rules invalidating test combinations

### 13. Collection Definition Errors in Expressions

```javascript
// WRONG — referencing collection before it's defined in the step chain
exp: 'collection.col_lcm.filter(lob=="Liability").map(lcm).first()'

// CORRECT — the collection reference should just be 'collection'
exp: 'collection.filter(lob=="Liability").map(lcm).first()'
```

The `collection` keyword in a collection step's format expressions always refers to that step's own `collection` data array.

## Handling Rate Ranges (Exposure Scoring)

Some products specify **ranges** rather than exact values for base premium or rate factors (e.g. "Low: $0.270–$0.629 per $1K"). These ranges make the model non-deterministic.

**Convert ranges into scored inputs** so the engine can compute a deterministic rate:

1. **Identify scoring dimensions** from the filing (e.g. complexity, financial stability, longevity)
2. **Create scored inputs** for each dimension (e.g. 5=Low risk, 3=Medium, 1=High)
3. **Aggregate to total score** via a calculation step
4. **Pre-compute interpolated rates** in the collection for each valid score
5. **Always include an override input** (e.g. `base_premium_override`, default 0) so users can bypass scoring when they have an exact value

```javascript
// Override escape hatch pattern
formula: '{{base_premium_override}} > 0 ? {{base_premium_override}} : {{computed_base}}'
```

## Exported Utilities

The package exports many utilities beyond the engine:

```javascript
const {
    // Core engine
    engine,

    // Query Language
    QL,

    // Expression & calculation helpers
    evaluateExpression,
    evaluateCalculation,
    evaluateFactors,
    evaluateExternal,
    evaluateCollection,
    evaluateCustomCode,
    evaluateExcelCode,
    excelFormulas,
    evaluateTests,
    generateTests,
    evaluateDiff,
    evaluateLists,
    evaluateURL,
    evaluateData,

    // Command mapping (UI helpers)
    commandToExpression,
    expressionToCommand,

    // Compression
    compressProject,
    decompressProject,
    isProjectCompressed,
    compressCollection,
    decompressCollection,

    // Schema validation
    evaluateSchema,
    buildFullSchema,
    projectSchema,
    inputSchema,
    outputSchema,
    // ... step-specific schemas

    // Templates
    bikeInsuranceTemplate,
    petInsuranceTemplate,

    // Prompt generation
    buildPrompt,

    // Test fixtures
    baseProject,
    testProject,
    testQuote,
} = require('@llow-group/swa_llow_pricing_engine');
```

## Project Compression

Projects can be compressed for storage/transfer. Collection data is the primary target — compressed format reduces file size significantly for large rate tables.

```javascript
const { compressProject, decompressProject, isProjectCompressed } = require('@llow-group/swa_llow_pricing_engine');

const compressed = compressProject(project);
const original = decompressProject(compressed);

// The engine accepts compressed projects directly
const result = await engine({ project: compressed, quote });
```

**Manual collection compression** (preferred for new models):

```javascript
// Uncompressed (readable but large)
collection: [
    { zip: '90001', territory: 'LA', rate: 247.00 },
    { zip: '90002', territory: 'LA', rate: 247.00 }
]

// Compressed (smaller, engine handles both)
collection: [
    ["zip", "territory", "rate"],
    ["90001", "LA", 247.00],
    ["90002", "LA", 247.00]
]
```

## Schema Validation

Every part of a project config has a corresponding JSON schema defined in `__schemas__/`. These are used to validate project structure before running the engine.

### `evaluateSchema`

The main validation function. Validates a full project (or an individual step) against its schema.

```javascript
const { evaluateSchema, testProject, testQuote } = require('@llow-group/swa_llow_pricing_engine');

// Validate a full project
const result = evaluateSchema({ project: myProject });
// { valid: true, errors: [] }

// Validate with input mapping check (pass a quote to test the mapping code runs)
const result = evaluateSchema({ project: myProject, quote: { age: 25 } });

// Validate a single step against its step schema
const { stepCalculationSchema } = require('@llow-group/swa_llow_pricing_engine');
const result = evaluateSchema({
    project: myStep,
    schema: stepCalculationSchema,
});
```

**Return value:**

```javascript
{
    valid: true | false,
    errors: [
        {
            path: 'input.format.name.label',  // JSON path to the problem
            keyword: 'required',               // Schema keyword that failed
            message: 'This is required',       // Human-readable message
            type: 'input' | 'output' | 'step', // Section type
            key: 'input',                       // Step key or section name
            step: 'transform',                  // Step type (for step errors)
        }
    ]
}
```

**What it validates:**

1. **Project structure** - `meta`, `input`, `steps`, and `output` are present with correct types
2. **Input schema** - `format` properties have required fields (`label`, `type`, `def`), property names match `^[A-Za-z_][A-Za-z0-9_]*$`
3. **Output schema** - `result` (with `formula`), `valid` (with `exp`), and `format` are valid
4. **Each step** - Validated against the step-specific schema (e.g. `calculation` requires `formula`, `exclusion` requires `items`)
5. **Input mapping** - If a `quote` is passed, the mapping code is executed to check for runtime errors
6. **Factors consistency** - `key` must match `result.key`, factor names cannot be empty
7. **Placeholder steps** - Steps with `placeholder: true` skip validation

### `buildFullSchema`

Returns the complete composed JSON schema with all step types inlined. This is useful for external tooling, editor autocompletion, or generating documentation.

```javascript
const { buildFullSchema } = require('@llow-group/swa_llow_pricing_engine');

const fullSchema = buildFullSchema();
// Returns a JSON Schema object with steps.items.oneOf containing all step schemas
```

The generated schema is also written to `dist/schema/index.json` when you run:

```bash
npm run generate:schema
```

### Available Schemas

Individual schemas are exported for validating specific sections or step types:

```javascript
const {
    // Top-level sections
    projectSchema,          // Full project structure
    inputSchema,            // Input block
    outputSchema,           // Output block
    testsSchema,            // Embedded test cases
    diffSchema,             // Diff/comparison
    processSchema,          // Process config

    // Step schemas (one per step type)
    stepTransformSchema,
    stepLabelSchema,
    stepLinksSchema,
    stepExclusionSchema,
    stepReferSchema,
    stepExcessesSchema,
    stepEndorsementsSchema,
    stepCalculationSchema,
    stepExternalSchema,
    stepFactorsSchema,
    stepCollectionSchema,
    stepCodeSchema,
    stepModularSchema,
    stepModulesSchema,
    stepBatchSchema,
} = require('@llow-group/swa_llow_pricing_engine');
```

Source schemas live in `__schemas__/` as JS modules. The full composed schema is generated to `dist/schema/index.json`.

## Insurance Rating Patterns

Common patterns observed across 50+ insurance model builds.

### Multiplicative Factor Chain

The most common rating algorithm in insurance. A base rate is multiplied by a chain of factors:

```
Total Premium = Base Rate × Territory × Age × Class × Limit × Deductible × ... × Round
```

```javascript
steps: [
    // 1. Look up base rate by coverage
    { step: 'collection', key: 'base_rates', collection: [/* ... */],
      format: { bi_base: { exp: 'collection.filter(coverage=="BI").map(rate).first()', def: 0 } } },

    // 2. Look up territory factor by ZIP
    { step: 'collection', key: 'territory', collection: [/* 2400 rows */],
      format: { territory_factor: { exp: 'collection.filter(zip={{zip_code}}).map(factor).first()', def: 1 } } },

    // 3. Look up age factor
    { step: 'collection', key: 'age_factors', collection: [/* ... */],
      format: { age_factor: { exp: 'collection.filter(min<={{driver_age}}).filter(max>={{driver_age}}).map(factor).first()', def: 1 } } },

    // 4. Calculate premium
    { step: 'calculation', key: 'bi_premium',
      formula: 'round({{bi_base}} * {{territory_factor}} * {{age_factor}}, 2)', def: 0 },

    // 5. Apply minimum premium floor
    { step: 'calculation', key: 'final_bi',
      formula: 'max({{bi_premium}}, {{minimum_premium}})', def: 0 }
]
```

### Per-Coverage Then Sum

Rate each coverage independently, then sum for total premium:

```javascript
steps: [
    // Rate BI, PD, COMP, COLL independently
    { step: 'calculation', key: 'bi_premium', formula: 'round({{bi_base}} * {{bi_factors}}, 2)' },
    { step: 'calculation', key: 'pd_premium', formula: 'round({{pd_base}} * {{pd_factors}}, 2)' },
    { step: 'calculation', key: 'comp_premium', formula: 'round({{comp_base}} * {{comp_factors}}, 2)' },

    // Sum all coverages
    { step: 'calculation', key: 'total_premium',
      formula: '{{bi_premium}} + {{pd_premium}} + {{comp_premium}}' }
]
```

### Optional Coverage Toggle

Enable/disable coverages based on user selection:

```javascript
// Input
{ key: 'comp_limit', type: 'string', def: 'no_coverage', items: ['no_coverage', '500', '1000'] }

// In formula — zero out premium when coverage not selected
formula: "({{comp_limit}} != 'no_coverage') ? round({{comp_base}} * {{comp_factors}}, 2) : 0"

// Exclusion rule for coverage dependencies
{ step: 'exclusion', items: [{
    name: 'GAP requires COMP and COLL',
    exp: '({{has_gap}} == true) and ({{comp_limit}} == "no_coverage" or {{coll_limit}} == "no_coverage")'
}]}
```

### Rounding Strategies

Different filings use different rounding approaches:

```javascript
// Strategy 1: Round at each step (most conservative, prevents compounding errors)
{ formula: 'round({{base}} * {{factor_1}}, 2)' },
{ formula: 'round({{step_1}} * {{factor_2}}, 2)' },

// Strategy 2: Round only the final premium (simplest)
{ formula: 'round({{base}} * {{f1}} * {{f2}} * {{f3}}, 0)' },

// Strategy 3: Round to nearest dollar at subtotals, nearest cent at final
{ formula: 'round({{bi_premium}}, 0)' },  // Coverage subtotal
{ formula: 'round({{total}} + {{fees}}, 2)' },  // Final with fees
```

Match the rounding strategy to what the filing specifies. A mismatch causes penny-level errors that compound across many factors.

## Validation Gates

When building models at scale (e.g. from insurance filings), the following validation gates catch the most common defects. Listed by failure frequency from production builds:

| Gate | Failure Rate | What It Catches |
|------|-------------|-----------------|
| Test Provenance | ~45% | No test uses inputs from the filing's worked example |
| Debug (Factor Variation) | ~40% | 80%+ of factor fields stuck at 1.0 (placeholder data) |
| Cohort | ~35% | Fewer than 30 valid test combinations generated |
| Collection Data | ~30% | Collection steps with empty or all-uniform values |
| Schema | ~15% | Missing required fields (`format`, `label`, etc.) |
| Product Identity | ~10% | Model name/type doesn't match the filing |
| Hardcode Detection | ~10% | Formulas embedding literal test answers |

Most models require 2-3 build attempts to pass all gates. The first attempt typically resolves schema and test provenance issues; the second fixes collection data and factor variation.

## Templates

Starter templates are included for common use cases:

```javascript
const { bikeInsuranceTemplate, petInsuranceTemplate } = require('@llow-group/swa_llow_pricing_engine');

const result = await engine({
    project: bikeInsuranceTemplate,
    quote: { /* ... */ },
});
```

## Development

```bash
npm test                    # run tests
npm run test:watch          # watch mode
npm run generate:types      # generate TypeScript definitions
npm run generate:schema     # generate JSON schemas
```
