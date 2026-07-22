# Module 3 Journal — PathReview

## Week 7 — Issue selection

**Issue link:** https://github.com/ascherj/pathreview/issues/148

**Issue title:** Skill extractor fails to detect JavaScript and TypeScript

**Tier:** [x] Tier 1  [ ] Tier 2  [ ] Tier 3

**Problem summary:**
PathReview's ingestion pipeline includes a `SkillExtractor`
(`ingestion/parsers/skill_extractor.py`) that scans resume and project text to
infer which technologies a candidate knows. Its language detection currently
recognizes JavaScript/TypeScript only through a narrow set of signals — an
`import`/`require` statement, a `.js`/`.ts` filename, or the literal string
`package.json` — so idiomatic code built on `const`/`let`/`var`, arrow
functions, `function` declarations, or `export`/`async` keywords is never
flagged. The class even defines a `JS_TS_KEYWORDS` set for exactly those
tokens, but `_detect_languages` never references it, leaving it as dead code;
the result is that text like "const arrow functions and async/await" returns no
skills, and TypeScript-specific syntax (interfaces, typed signatures) is missed
or downgraded to plain JavaScript. A successful fix would wire the JS/TS
keyword and TypeScript-syntax signals into `_detect_languages` so genuine
JavaScript and TypeScript usage is detected with an appropriate confidence
score, which is verified by the four currently-failing unit tests in
`tests/unit/test_skill_extractor.py` (`test_javascript_detection`,
`test_text_with_typescript_files`, `test_devops_tool_detection`, and
`test_docker_compose_detection`).

**"Is this right for me?" — selection notes:**
- **Tier fit:** Labeled `tier-1` / good-first-issue and estimated at a few
  hours. This is my first contribution to a large codebase, so a Tier 1 bug is
  the right starting point (rather than a Tier 2/3 feature that spans modules).
- **Scope is contained:** The fix lives in a single file
  (`ingestion/parsers/skill_extractor.py`) and one detection method
  (`_detect_languages`). No API, schema, or frontend changes are required, so
  there's little risk of scope creep into other modules.
- **"Done" is already defined:** The issue names four failing tests in
  `tests/unit/test_skill_extractor.py`. That gives me an objective,
  test-driven finish line instead of an open-ended judgment call.
- **I understand the root cause:** The bug is concrete — a `JS_TS_KEYWORDS`
  constant that is defined but never used — so I can explain both what's broken
  and why, which lowers the risk of getting stuck in Weeks 8–9.
- **Reproducible:** The issue provides exact input strings and expected
  outputs, so I can reproduce the failure locally before touching any code.

**Branch name:** `fix/148-skill-extractor-js-ts-detection`

**Local run confirmation:** [ ] App runs locally at localhost:5173
*(Pending — Docker Desktop must be started, then `make setup` && `make run`.)*

**Cohort ledger:** [ ] Issue added to cohort ledger

## Week 8 — Reproduction & solution planning

**Reproduction commit link:**
https://github.com/Spectre1404/pathreview/commit/232155edb01c956f1cc65002107dab1117e1dd32

**Reproduction summary:**
Ran the existing `SkillExtractor` on the exact inputs from the four failing unit
tests via `PYTHONPATH=. python -m pytest tests/unit/test_skill_extractor.py -q`.
Idiomatic JS detected nothing, TypeScript and a Dockerfile were both mislabeled as
Python (0.70), and a docker-compose file detected nothing — while the same JS body
with a `.js` filename was detected, confirming the bug is the missing keyword/syntax
signals.

**PLAN.md link:**
https://github.com/Spectre1404/pathreview/blob/fix/148-skill-extractor-js-ts-detection/PLAN.md

**Walkthrough video (recommended):** Not recorded (optional, not graded).

**Blockers or open questions:**
`make check` / `make test-unit` can't run locally yet — the full `tests/unit` suite
fails collection on missing deps (e.g. `structlog`) because `make setup`/`.venv`
isn't done (needs Docker Desktop). I verify against the single target test file for
now and will attempt `make setup` before the Week 9 PR. Open decision: whether to
also fix the pre-existing typo in `test_database_technology_detection` (line 138),
which is unrelated to #148.

### Reproduction detail

**Status:** Reproduced reliably and confirmed the root cause. No fix applied yet.

**Where the bug lives:** `ingestion/parsers/skill_extractor.py`, method
`_detect_languages` (JS/TS block at lines ~173–192). The class defines a
`JS_TS_KEYWORDS` set at line 30 that `_detect_languages` never references — dead
code. JS/TS detection fires only on a `.js`/`.ts` filename, `import`/`require`, or
the literal `package.json`.

**How to reproduce (built-in, no extra files):**

```bash
PYTHONPATH=. python -m pytest tests/unit/test_skill_extractor.py -q
```

Result: 4 in-scope tests fail — `test_javascript_detection`,
`test_text_with_typescript_files`, `test_devops_tool_detection`,
`test_docker_compose_detection`. (A 5th failure,
`test_database_technology_detection`, is a pre-existing typo in the test itself —
line 138 reads `skill_names = [s.name for s in skill_names]`, using the variable
before it is defined. Out of scope for #148.)

**Observed behavior — actual vs. expected** (extractor run on the exact test inputs):

| Case | Detected now | Should be |
|------|--------------|-----------|
| Idiomatic JS (`const`/`require('fs')`/`console.log`) | nothing `[]` | JavaScript |
| TypeScript (`interface`, `id: string`, `Promise<User>`) | **Python (0.70)** | TypeScript |
| Dockerfile (`FROM`/`RUN pip install`/`EXPOSE`) | **Python (0.70)** | Docker |
| docker-compose (`version:`/`services:`/`ports:`) | nothing `[]` | Docker |
| Control: same JS body but filename `app.js` | JavaScript (0.70) | (narrow path works) |

The control row confirms the diagnosis: strip the `.js` filename and identical code
detects nothing — detection depends entirely on the narrow filename/literal signals.

**Root-cause mechanisms confirmed (each traced to a specific line):**

1. **Dead constant.** `JS_TS_KEYWORDS` (line 30) is never used by `_detect_languages`,
   so `const`/`let`/`var`/`function`/`export`/`async` never contribute a signal.
2. **`require(...)` even slips the narrow path.** The one ES/CJS check,
   `re.search(r"\b(import|require)\s+", text)` (line 179), requires whitespace after
   `require`, but idiomatic `require('fs')` has `(` immediately — no match. Hence the
   first case detects literally nothing.
3. **TypeScript is mislabeled as Python, not merely missed.** The Python
   type-annotation regex `:\s*(int|str|float|bool|list|dict)` (line 160) matches
   `id: string` because `string` starts with `str`, scoring the TS sample as
   Python (0.70).
4. **Docker inputs never contain the literal token `docker`.** `_detect_tools`
   (line 267) does a substring match on `"docker"`; a real Dockerfile/compose file
   mentions neither, so one is misread as Python (via `requirements.txt`, line 162)
   and the other detects nothing.

**Scope note:** two of the four target tests (`test_devops_tool_detection`,
`test_docker_compose_detection`) are about Docker/DevOps detection, not JS/TS. Issue
#148's title is JS/TS-specific but lists all four as the "done" criteria, so the fix
is broader than the title implies. Will address in PLAN.md.
