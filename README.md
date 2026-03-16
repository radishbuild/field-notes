# field-notes

Technical investigations, learnings, and write-ups.

## Contents

- [**gemini-any-mode-enum-bug**](gemini-any-mode-enum-bug/) — Gemini ANY mode silently rejects tool schemas when enum values exceed an opaque complexity budget. Includes a minimal repro notebook.
- [**gemini-maxitems-cumulative-budget**](gemini-maxitems-cumulative-budget/) — Gemini rejects tool schemas when the cumulative sum of `maxItems` across all properties exceeds ~960. Affects both ANY and AUTO modes. Includes a minimal repro notebook.
