# GitHub Issue Draft

**Repo:** `googleapis/python-genai`

---

**Title:** INVALID_ARGUMENT in ANY mode when tool declaration payload exceeds undocumented limit (~77 KB); AUTO mode passes with identical tools

---

This is a **product issue** — the API server rejects valid tool schemas with an opaque error. The SDK passes the schema through correctly; the rejection comes from the Gemini API itself. Filing here since the SDK is the interface and there's no public product issue tracker.

#### Environment details

- Programming language: Python 3.11+
- OS: Any (reproduced on Colab, macOS, Linux)
- Language runtime version: Python 3.11
- Package version: `google-genai` latest (tested with 1.x)
- Model: `gemini-3-flash-preview`

#### Steps to reproduce

1. Open the self-contained Colab notebook: [Open in Colab](https://colab.research.google.com/github/radishbuild/field-notes/blob/main/gemini-tool-declaration-size-limit/repro_anonymized.ipynb) · [Source on GitHub](https://github.com/radishbuild/field-notes/blob/main/gemini-tool-declaration-size-limit/repro_anonymized.ipynb)
2. Set your `GEMINI_API_KEY` in Colab Secrets
3. Run all cells

The notebook loads **real-world tool schemas** from a production agent platform (names/descriptions anonymized, schema shapes preserved exactly) via a companion JSON file. It includes 95 tools spanning code management, email, calendar, messaging, CRM, scraping, and media operations — a realistic configured agent.

#### What happens

When using function calling with `mode="ANY"` (forced function calling), the API returns `INVALID_ARGUMENT` with no diagnostic information when the total tool declaration payload exceeds an undocumented threshold (~77 KB of serialized JSON).

**The same request with `mode="AUTO"` and identical tools passes.** The limit is specific to ANY mode.

This is distinct from the previously reported [maxItems cumulative budget](https://github.com/googleapis/python-genai/issues/XXX) and [enum complexity](https://github.com/googleapis/python-genai/issues/XXX) issues — no `maxItems`, no enums are involved.

#### Key observations

1. **ANY mode fails, AUTO mode passes.** With 95 tools (~77 KB), ANY mode returns `INVALID_ARGUMENT` while AUTO mode succeeds with the exact same tool declarations, system prompt, and user message.

2. **Total payload size is the budget.** Removing tools until the total size drops below the threshold makes ANY mode pass again. The threshold tracks cumulative declaration size, not tool count.

3. **A small delta can tip the balance.** Swapping one smaller tool for a slightly larger one can move the request from pass to fail in ANY mode.

4. **System prompt and conversation history are irrelevant.** The failure reproduces with a minimal system prompt (`" "`) and a single-word user message (`"hi"`). Tool declarations alone are sufficient to trigger it.

5. **Error is completely opaque via the API.** The SDK response is a bare `INVALID_ARGUMENT` with no actionable information. AI Studio's UI occasionally surfaces the internal error:

   ```
   Constraint is too tall: 6336 (vs max of 5888)
   ```

   This reveals an internal "constraint height" budget (max 5888 units) but the units are undocumented — we don't know how they map to tool count, schema size, property count, or any other observable metric. There is no way for API consumers to predict or validate whether a tool set will exceed this budget before sending the request.

#### Expected behavior

1. The API should either accept these schemas in ANY mode or return a descriptive error indicating what exceeds the limit, what the limit is, and how much the request needs to be reduced.

2. If ANY mode has a lower tool declaration budget than AUTO mode, this should be documented so developers can design schemas accordingly and validate before sending requests.

3. The limit should be high enough to support realistic agent workflows with 60-100 tools — a common requirement for platforms that integrate with email, calendars, code management, CRM, and other API-heavy domains. ANY mode is the standard choice for agent platforms that need guaranteed tool usage.

#### Workarounds

- **Switch to AUTO mode** — higher limit, but loses the guarantee that the model will call a tool
- **Reduce tool schema size** — trim descriptions, remove optional properties, simplify nested schemas before sending to Gemini
- **Limit concurrent tool count** — dynamically select a subset of tools per request based on context

#### Production impact

This is a real-world issue for agent platforms that use ANY mode (forced function calling) and dynamically assemble tool sets. A typical integration with 95 tools (email, calendar, GitHub, Slack, CRM, scraping, media generation) produces ~77 KB of tool declarations — right at the ANY-mode boundary. Adding or swapping a single tool can silently break the entire agent, while the same tool set works fine in AUTO mode.

---
