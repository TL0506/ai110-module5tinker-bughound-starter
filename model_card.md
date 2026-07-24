# BugHound Mini Model Card (Reflection)

Fill this out after you run BugHound in **both** modes (Heuristic and Gemini).

---

## 1) What is this system?

**Name:** BugHound
**Purpose:** Analyze a Python snippet, propose a fix, and run reliability checks before suggesting whether the fix should be auto-applied.

**Intended users:** Students learning agentic workflows and AI reliability concepts.

---

## 2) How does it work?

`BugHoundAgent.run()` executes a fixed 5-step loop (`bughound_agent.py`):

1. **PLAN** — logs intent, no real decision made here.
2. **ANALYZE** — detects issues. If no client is attached (`client=None` / Heuristic mode), uses three hardcoded regex/string rules: bare `except:` (High/Reliability), `print(` (Low/Code Quality), `TODO` (Medium/Maintainability). If a Gemini client is attached, it loads `prompts/analyzer_system.txt` + `analyzer_user.txt`, sends the code, and expects a JSON array of `{type, severity, msg}` back. If the response isn't parseable JSON or the call throws, it silently falls back to the heuristic rules above.
3. **ACT** — proposes a fix. No issues → code returned unchanged. Heuristic mode does simple regex replacement (bare except → `except Exception as e:`, `print(` → `logging.info(`). Gemini mode sends the issues + code to the fixer prompt, expects the full rewritten file back with no markdown fences, and falls back to the heuristic fixer on empty output or an exception.
4. **TEST** — `assess_risk()` (in `reliability/risk_assessor.py`) scores the diff 0-100 using issue severities plus structural checks (code shrank a lot, `return` disappeared, print issue reported but not fixed).
5. **REFLECT** — `should_autofix = (risk level == "low")`. If not low, the agent logs that human review is required; it never applies the fix itself either way — the UI just displays the recommendation.

---

## 3) Inputs and outputs

**Inputs tested:**

- `sample_code/cleanish.py` — a clean 5-line function with logging already in place, no issues expected.
- `sample_code/flaky_try_except.py` — a `load_text_file` function with a bare `except:` and an unclosed file handle.
- `sample_code/mixed_issues.py` — a function combining a TODO comment, a `print(...)` call, and a bare `except:` (all three heuristic rules at once), run twice in Gemini mode to compare outputs.

Shapes: all single-function snippets, 5-10 lines, centered on error handling and I/O — the kind of "toy but realistic" snippet where heuristics and an LLM would both plausibly notice the same things.

**Outputs observed:**

- Heuristic mode: exactly the 3 canned issue types above, deterministic every run.
- Gemini mode on `mixed_issues.py`: 3 issues (TODO/Low, print/Low, bare-except/High), matching the heuristic categories but with model-written `msg` text and free-text `type` labels (e.g. "Bare Except", "Correctness", "Debugging Statement" — not a fixed vocabulary).
- Risk reports: `flaky_try_except.py` scored 55/medium; `mixed_issues.py` scored 45/medium on both Gemini runs. Both correctly resulted in `should_autofix: false` and a "Human review recommended" REFLECT log.

---

## 4) Reliability and safety rules

**Rule 1 — Severity-based score deduction** (`for issue in issues: severity == "high"/"medium"/"low" → -40/-20/-5`).
- *Why it matters:* ties the risk score directly to how serious the detected issues are, so a fix addressing a bare `except:` is treated as riskier than one addressing a stray `print`.
- *False positive:* a High-severity issue whose fix is actually trivial and safe (e.g. renaming one bare `except:` to `except Exception:`) still gets a harsh -40, potentially blocking an autofix that was fine.
- *False negative (the one we found and patched):* this check does an exact string match on `"high"/"medium"/"low"`. Before our fix, an LLM-issued severity like `"critical"` or a typo would match none of the three branches and silently contribute **zero** deduction — understating risk instead of failing safe.

**Rule 2 — `should_autofix = (level == "low")`.**
- *Why it matters:* single, simple gate — no matter how the score was computed, only the safest tier is allowed to auto-apply.
- *False positive (blocks a safe fix):* a fix that only touches a Low-severity issue but happens to also shrink the code by >50% (e.g. removing a large commented-out block) gets sent to "medium" by the structural check even though the actual behavior is unchanged.
- *False negative (lets a risky fix through):* the score is purely about *what was reported*, not whether the fix actually did what it claimed. A fix that quietly leaves a reported issue unresolved (see failure mode below) could still land in the "low" band if the untouched issue was Low severity — which is exactly why we added the new print-not-resolved signal.

---

## 5) Observed failure modes

1. **Fixer silently drops an issue instead of fixing it.** Running Gemini mode on `mixed_issues.py`, the analyzer correctly reported all 3 issues (TODO, print, bare except), but the first fixer response returned code with the bare-except fixed and the `print("computing ratio...")` line and the TODO comment simply *deleted* — not converted to logging, not explained, just gone. Nothing in the agent or risk assessor previously checked whether a reported issue was actually addressed vs. silently removed/ignored.
2. **Fix scope varies run to run and can grow beyond the reported issues.** Running the same file (`mixed_issues.py`) through Gemini mode a second time (same temperature, same prompt) produced a structurally different fix: it introduced a brand-new `validate_input` helper function and changed the caught exception types from `except:` to `except (ZeroDivisionError, TypeError):`. Both are reasonable improvements, but neither was one of the 3 reported issues — the fixer is not constrained to *only* address what was analyzed, so "minimal diff" is aspirational (per the prompt text) rather than enforced.

---

## 6) Heuristic vs Gemini comparison

- **Detection overlap:** on `mixed_issues.py`, Gemini found the same 3 categories the heuristics would have (TODO, print, bare except) — heuristics and Gemini agreed on *what* to flag for this file.
- **What Gemini adds:** richer, issue-specific `msg` text (e.g. explaining *why* bare except is dangerous — masks `SystemExit`/`KeyboardInterrupt`) and, on `flaky_try_except.py`, it caught a second issue heuristics have no rule for at all: the file handle not being closed via a context manager on the exception path.
- **What heuristics catch consistently:** the exact 3 keyword/regex patterns, every time, with zero variance — useful as a stable fallback but blind to anything not literally `print(`, bare `except:`, or `TODO`.
- **Fix differences:** heuristic fixes are narrow, mechanical, and identical every run. Gemini fixes vary run-to-run (see failure mode #2) and can restructure code beyond the reported issues.
- **Risk scorer agreement:** yes — in both Gemini runs the scorer landed on "medium" and correctly refused autofix, matching my intuition that a bare-except fix touching error-handling logic shouldn't auto-merge without a human looking at it.

---

## 7) Human-in-the-loop decision

**Scenario:** the LLM fixer reports 3 issues but the returned "fixed" code still exhibits behavior tied to one of those reported issues (e.g. a print/logging issue was reported but `print(` is still present in the output, or a resource-leak issue was reported but the file is still opened without a context manager). This means the fixer's own output contradicts its own analysis — either it forgot to fix something or the JSON/prose mismatch is confusing it.

- **Trigger:** the fixed code still contains a code pattern tied to an issue category that was reported (implemented as the new print-check below; the same pattern generalizes to other issue types).
- **Where implemented:** in `reliability/risk_assessor.py`'s `assess_risk`, since it's a pure content-based check on `(original_code, fixed_code, issues)` with no side effects — the natural home for "does this fix actually look consistent with what was reported," alongside the existing structural checks.
- **Message shown:** the existing REFLECT log line ("Fix is not safe enough to auto-apply. Human review recommended.") already covers this; the new reason string surfaced in the risk report specifically says *"A print-statement issue was reported but not resolved in the fix."* so a reviewer knows exactly what to double check.

---

## 8) Improvement idea

**Implemented this session:** two small, targeted changes.

1. **Severity validation at parse time** (`bughound_agent.py::_normalize_issues`): any severity string that isn't exactly `low`/`medium`/`high` (case-insensitive) is now coerced to `"High"` instead of being passed through as an arbitrary string. Previously, an LLM typo or new label would silently score as zero risk in `assess_risk`'s exact-match branches — a fail-open behavior that could unblock an unsafe autofix. Defaulting unknown severities to the most cautious tier makes the failure mode fail-*closed* instead.
2. **New risk signal: unresolved print issue** (`reliability/risk_assessor.py::assess_risk`): if any reported issue mentions "print" and the fixed code still contains `print(`, the score drops by 15 and a reason is logged. This directly targets the observed failure mode (#1 above) where the fixer's output silently left a reported issue unaddressed. Both are covered by new offline tests (`tests/test_agent_workflow.py::test_analyze_normalizes_unknown_severity_to_high`, `tests/test_risk_assessor.py::test_unresolved_print_issue_blocks_autofix`) that don't consume API quota.

**Next idea (not yet implemented):** generalize signal #2 into a small table of `{issue keyword → code pattern that should have disappeared}` (print→`print(`, bare-except→`except:`, resource-leak→missing `with open`) so any reported-but-unaddressed issue is caught, not just print statements.
