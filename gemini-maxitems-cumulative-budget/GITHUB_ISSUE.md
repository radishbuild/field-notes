# GitHub Issue Draft

**Repo:** `googleapis/python-genai`

---

**Title:** INVALID_ARGUMENT when cumulative sum of `maxItems` across tool schema properties exceeds ~960

---

### Environment

- API: Gemini API via `google-genai` Python SDK
- Model: `gemini-3-flash-preview`
- SDK version: latest (`pip install google-genai`)

### Description

When using function calling with `parametersJsonSchema`, the API returns `INVALID_ARGUMENT` with no diagnostic information when the **cumulative sum of `maxItems` values** across all properties in a tool schema exceeds an undocumented budget of approximately 960. This affects **both ANY and AUTO modes**.

Other JSON Schema numeric constraints (`maximum`, `minimum`, `maxLength`) at identical values are completely unaffected.

### Reproduction

Self-contained Colab notebook — open, set API key, run all cells:

[Open in Colab](https://colab.research.google.com/github/radishbuild/field-notes/blob/main/gemini-maxitems-cumulative-budget/repro_minimal.ipynb) · [Source on GitHub](https://github.com/radishbuild/field-notes/blob/main/gemini-maxitems-cumulative-budget/repro_minimal.ipynb)

The notebook generates a single tool with N array properties, each with a configurable `maxItems` value. It sweeps across different field counts and `maxItems` values to isolate the cumulative budget.

### Results

**Single field — boundary sweep:**

```
1 field × maxItems=N  (sum = N)
  maxItems  │     ANY    AUTO
──────────────────────────────────
       971  │  PASS ✓  PASS ✓
       972  │  FAIL ✗  FAIL ✗
```

**8 fields — cumulative sum matters:**

```
8 fields × maxItems=N  (sum = 8×N)
  maxItems    sum  │     ANY    AUTO
────────────────────────────────────────
       119    952  │  PASS ✓  PASS ✓
       120    960  │  FAIL ✗  FAIL ✗
```

**Different distributions, same sum — all behave consistently:**

```
Fields × maxItems =   Sum  │     ANY    AUTO
──────────────────────────────────────────────
Under budget (~950):
     1 ×      950 =   950  │  PASS ✓  PASS ✓
    10 ×       95 =   950  │  PASS ✓  PASS ✓
    50 ×       19 =   950  │  PASS ✓  PASS ✓

Over budget (~980):
     1 ×      980 =   980  │  FAIL ✗  FAIL ✗
    10 ×       98 =   980  │  FAIL ✗  FAIL ✗
    50 ×       20 =  1000  │  FAIL ✗  FAIL ✗
```

**Control — stripping `maxItems` fixes it:**

```
Same failing schemas with maxItems removed → all PASS ✓
```

**Cross-keyword comparison (20 fields × value=200, sum=4000):**

```
     Keyword  │     ANY    AUTO
──────────────────────────────
    maxItems  │  FAIL ✗  FAIL ✗
     maximum  │  PASS ✓  PASS ✓
     minimum  │  PASS ✓  PASS ✓
   maxLength  │  PASS ✓  PASS ✓
```

### Key observations

1. **Both ANY and AUTO modes.** Unlike the enum complexity issue (which only affects ANY mode), this budget applies to both calling modes.

2. **Cumulative sum is the budget.** The threshold is ~960 regardless of how the `maxItems` values are distributed across properties. 1 field with `maxItems=972` fails, as do 8 fields with `maxItems=120` each (sum=960).

3. **Only `maxItems` is affected.** `maximum`, `minimum`, and `maxLength` at identical or even larger cumulative values all pass. The budget is specific to `maxItems`.

4. **Error is completely opaque.** The response is `INVALID_ARGUMENT` with no indication of which property, what the limit is, or that `maxItems` is the cause.

5. **Production impact.** Tool schemas generated from OpenAPI specs commonly have many array fields with `maxItems` constraints. A typical CRM search tool with 20 array fields each having `maxItems: 100` (sum=2000) will silently fail.

### Expected behavior

1. The API should either accept these schemas or return a descriptive error indicating what exceeds the limit, which property is responsible, and what the limit is.

2. If a cumulative budget on `maxItems` is intentional, it should be documented so developers can design schemas accordingly.

### Workaround

- Strip `maxItems` and `minItems` from tool schemas before sending to Gemini, and validate array length constraints in the application layer instead
- If `maxItems` must be included, ensure the cumulative sum across all properties stays well under ~960

---
