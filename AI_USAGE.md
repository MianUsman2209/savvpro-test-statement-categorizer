# AI Usage Report

## 1. AI Tools Used
- **Tool / Environment:** Google Antigravity Agentic IDE
- **Core Model:** Gemini 3.8 Flash

---

## 2. How AI Helped
1. **Full-Stack Architecture & Scaffolding:** Designed a modular, production-ready structure separating FastAPI backend layers (`api/`, `engine/`, `services/`, `models/`) from the React + TypeScript frontend.
2. **Deterministic Brain Engineering:** Guided the construction of the multi-layered categorization engine, adhering strictly to the constraint of **zero AI/LLM API calls** for classification decisions.
3. **Comprehensive Normalization Regexes:** Formulated regular expressions to strip noisy bank transaction markers (POS prefixes, store identifiers, payment aggregators, terminal codes, embedded dates, and geographic tokens).
4. **Edge-Case Modeling:** Designed rule definitions for refund/reversal linking, duplicate charge detection, non-spending internal transfers, and personal vs. business ambiguity gating.
5. **Automated Test Suite Creation:** Generated 28 backend unit/integration tests with `pytest` and frontend tests with `vitest` + React Testing Library, validating parser edge cases, rules, and error handling.

---

## 3. Specific Mistakes Made by the AI and How They Were Resolved

### Mistake 1: Thread Lock Self-Deadlock in `RuleStore`
- **Issue:** During unit test execution for active rule learning, `test_user_rule_persistence_and_precedence` hung indefinitely.
- **Root Cause:** In `backend/app/services/rule_store.py`, `add_rule()` acquired `with self._lock:` and subsequently called `self.get_all()`, which also attempted to acquire `with self._lock:`. Because `threading.Lock()` is non-reentrant, the thread deadlocked against itself.
- **Resolution:** Replaced `from threading import Lock` with `from threading import RLock` and instantiated `self._lock = RLock()`, allowing safe re-entrant acquisition within the same thread.

### Mistake 2: Greedy Location Regex Stripping Single-Word Merchants
- **Issue:** In `backend/app/engine/normalizer.py`, descriptions like `"AWS 2024-08-01 SEATTLE WA US"` were normalized into an empty string `""`.
- **Root Cause:** The trailing location pattern `[A-Z]{2,15}\s+[A-Z]{2,15}\s+({US_STATES})` matched `"AWS"` as the first city word, `"SEATTLE"` as the second city word, and `"WA"` as the state code, consuming the entire string.
- **Resolution:** Introduced a lookbehind `(?<=\s)[A-Z]{2,15}(?:\s+[A-Z]{2,15})?\s+(?:{US_STATES})` requiring whitespace before the city name. This ensures that the leading merchant token is preserved.

### Mistake 3: Unsigned Single-Amount Column Classification
- **Issue:** In `backend/app/services/parser.py`, statements with a single `Amount` column containing negative expenses and positive deposits (`STRIPE PAYOUT 3200.00`) were misclassified as `DEBIT`.
- **Root Cause:** The parser applied a default assumption suitable for positive-only credit card statements (where positive numbers represent charges).
- **Resolution:** Enhanced `StatementParser` to scan the initial rows of the dataset: if negative numbers (`-` or `(...)`) are present in a single Amount column, positive values are recognized as `CREDIT` (incoming deposits), while negative values are treated as `DEBIT` (expenses).

### Mistake 4: Rapidfuzz Ratio Selection for Extended Descriptions
- **Issue:** In `backend/app/engine/brain.py`, the test `test_fuzzy_match_rapidfuzz` failed when testing `"JETBRAINSS CLOUD DEV"` against canonical merchant `"JETBRAINS"`.
- **Root Cause:** `fuzz.token_set_ratio` produced a score of only ~62% due to extra descriptive tokens.
- **Resolution:** Switched to evaluating `max(fuzz.WRatio(norm_desc, canonical_name), fuzz.token_set_ratio(norm_desc, canonical_name))`. Rapidfuzz's `WRatio` properly scored the match at 90%, triggering the fuzzy layer above the 82% threshold.

---

## 4. Token usage

Development was conducted within the Google Antigravity Agentic IDE using the Gemini 3.8 Flash model. The session operated on interactive conversational turns with full local tool assistance (environment diagnostics, test execution, and code generation). In adherence to the prompt guidelines ("do not invent token numbers"), precise external API billing counts are omitted as standard IDE sessions run without an exposed external per-token billing meter. All tool invocations and reasoning steps were executed live during the pair-programming session.
