# AI Usage Report

> **Compliance with "The One Big Rule":**
>
> *"The brain must be your own code. Do not use a big AI model (like GPT, Claude or Gemini) to decide a category. Your code decides what each transaction is and when it is not sure. You can use AI tools to help you write the code. We want to see how you use them."*
> — **TASK.md**
>
> **Declaration of Adherence:**
> - **Runtime Categorization Engine ("The Brain"):** Contains **ZERO AI / LLM calls or API dependencies**. Neither OpenAI, Anthropic, nor Gemini are imported or called. Every category, confidence score, and decision reason is generated through 100% deterministic Python code (sanitization regexes, dictionary lookups, Rapidfuzz fuzzy algorithms, and calibrated threshold gates).
> - **Development Assistance:** AI was used exclusively as an IDE pair-programmer to assist in writing, testing, and debugging the application code, as permitted by the assessment guidelines.

---

## 1. AI Tools Used
- **IDE / Environment:** Google Antigravity Agentic Coding Assistant
- **Model Used for Code Generation:** Gemini 3.8 Flash

---

## 2. How AI Assisted in Writing the Code

AI was used as an interactive pair-programming assistant across the following areas:

1. **Full-Stack Scaffolding:** Generated the initial layered boilerplate for the FastAPI backend (`api/`, `engine/`, `services/`, `models/`) and React + TypeScript frontend dashboard.
2. **Deterministic Brain Engineering:** Helped draft the rule matrices, category enums, and deterministic scoring math without relying on external machine learning services.
3. **Regular Expression Drafting:** Formulated initial regex patterns to sanitize bank statement descriptions by stripping store numbers (`#\d+`), POS prefixes (`CHECKCARD`, `DEBIT CARD`), payment aggregators (`SQ *`, `TST*`), embedded dates, and geographic tokens.
4. **Edge-Case Modeling:** Assisted in structuring logic for linking expense refunds against original debits, isolating non-spending internal transfers, detecting duplicate charges, and gating ambiguous mixed-use vendors (Amazon, Target, Costco) to "Needs Review".
5. **Comprehensive Test Suite Generation:** Created test fixtures and edge-case scenarios covering 28 backend `pytest` tests and frontend `vitest` tests.

---

## 3. Specific Mistakes Made by the AI and How They Were Resolved

Human oversight caught several critical implementation bugs introduced during code generation:

### Mistake 1: Thread Lock Self-Deadlock in `RuleStore`
- **Issue:** When running `test_user_rule_persistence_and_precedence`, the test suite hung indefinitely.
- **Root Cause:** In `backend/app/services/rule_store.py`, the AI implemented `add_rule()` with `with self._lock:` and then immediately called `self.get_all()`, which also attempted to acquire `with self._lock:`. Because the AI imported standard `threading.Lock()` (which is non-reentrant), the thread deadlocked against itself.
- **Resolution:** Caught through terminal process inspection. Replaced `from threading import Lock` with `from threading import RLock` and initialized `self._lock = RLock()`, permitting safe re-entrant acquisition within the same thread.

### Mistake 2: Overly Greedy Location Regex Consuming Merchant Names
- **Issue:** In `backend/app/engine/normalizer.py`, descriptions like `"AWS 2024-08-01 SEATTLE WA US"` were reduced to an empty string `""`.
- **Root Cause:** The AI generated a regex `[A-Z]{2,15}\s+[A-Z]{2,15}\s+({US_STATES})` to strip trailing city and state tokens. However, without a leading boundary constraint, the pattern treated `"AWS"` as the city name, `"SEATTLE"` as the second city token, and `"WA"` as the state, swallowing the entire merchant name.
- **Resolution:** Refactored the regex to include a positive lookbehind `(?<=\s)[A-Z]{2,15}(?:\s+[A-Z]{2,15})?\s+(?:{US_STATES})` requiring whitespace before the city name. This ensures standalone merchant identifiers (like `AWS`) are preserved.

### Mistake 3: Single-Amount Column Sign Convention Flaw
- **Issue:** On the clean sample statement (`clean_business_credit_card.csv`), the parser misclassified all purchase debits as credits, incorrectly placing them under `Refunds & Reversals`.
- **Root Cause:** The AI attempted to detect signed bank statements using `if "-" in line:` across the entire CSV row text. Because date strings (`2024-03-01`) and phone numbers (`800-826-0428`) contained hyphens, `has_negative_numbers` evaluated to `True` for every row, triggering the deposit branch.
- **Resolution:** Replaced the line-wide check with a column-specific inspection: `val = row[amt_idx].strip(); if val.startswith("-") or val.startswith("(") or "DR" in val:`. This properly distinguished unsigned credit card purchase charges (debits) from signed bank statements.

### Mistake 4: Fuzzy Metric Selection in Rapidfuzz
- **Issue:** In `backend/app/engine/brain.py`, the fuzzy test `test_fuzzy_match_rapidfuzz` failed for `"JETBRAINSS CLOUD DEV"` against `"JETBRAINS"`.
- **Root Cause:** The AI initially selected `fuzz.token_set_ratio`, which scored only ~62% due to length and token disparity penalties, failing the 82% threshold.
- **Resolution:** Updated the fuzzy engine to evaluate `max(fuzz.WRatio(norm_desc, canonical_name), fuzz.token_set_ratio(norm_desc, canonical_name))`. `WRatio` properly weighted the substring similarity at 90%, allowing the fuzzy fallback layer to correctly categorize the vendor.

---

## 4. Token Usage

Development was conducted within the Google Antigravity Agentic IDE using the Gemini 3.8 Flash model. The session operated on interactive conversational turns with full local tool assistance (environment diagnostics, test execution, and code generation). In adherence to the prompt guidelines ("do not invent token numbers"), precise external API billing counts are omitted as standard IDE sessions run without an exposed external per-token billing meter. All tool invocations and reasoning steps were executed live during the pair-programming session.
