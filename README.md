# field-notes

Technical investigations, learnings, and write-ups.

## Contents

- [**gemini-any-mode-enum-bug**](gemini-any-mode-enum-bug/) — Gemini ANY mode silently rejects tool schemas when enum values exceed an opaque complexity budget. Includes a minimal repro notebook.
- [**gemini-maxitems-cumulative-budget**](gemini-maxitems-cumulative-budget/) — Gemini rejects tool schemas when the cumulative sum of `maxItems` across all properties exceeds ~960. Affects both ANY and AUTO modes. Includes a minimal repro notebook.
- [**gemini-tool-declaration-size-limit**](gemini-tool-declaration-size-limit/) — Gemini ANY mode (forced function calling) rejects requests when the total tool declaration payload exceeds ~77 KB; AUTO mode passes with identical tools. Includes a repro notebook with anonymized tool schemas.
