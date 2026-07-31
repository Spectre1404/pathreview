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

## Week 9 — Solution building & PR submission

### Check-in 1 (mid-week)

**Current progress:**
Implemented the fix in `ingestion/parsers/skill_extractor.py`, completing PLAN.md
sub-tasks 2–5: (2) JavaScript keyword/arrow-function detection plus a corrected
`require(` matcher; (3) TypeScript disambiguation via `interface`/`enum`/type-alias/
typed-signature syntax; (4) tightened the Python annotation regex with a word
boundary so `: string` no longer scores as Python; (5) Docker detection from
Dockerfile directives and docker-compose structure in `_detect_tools`. All four
target tests now pass, and I added four regression tests. Baselined `make test-unit`
(53 failed / 375 passed) and `make check` (182 pre-existing ruff errors) on a clean
checkout first.

**Next steps:**
Open the PR from the fork branch into `main`, fill in the PR template, request peer
feedback in Slack, and finalize.

**Blockers:**
The repo has substantial pre-existing failures unrelated to #148 (documented in the
PR's Notes for Reviewers). Treating "passes" as "introduces no new failures," per the
assignment guidance. Local `make setup` DB/frontend steps still need Docker, but unit
tests and lint/typecheck run without it once `.venv` is created.

---

### Check-in 2 (end of week)

**PR link:** https://github.com/Spectre1404/pathreview/pull/1

**Branch:** `fix/148-skill-extractor-js-ts-detection`

**What you built:**
Wired the previously-unused `JS_TS_KEYWORDS` signal and TypeScript-syntax detection
into `_detect_languages`, and added Docker detection (Dockerfile directives +
docker-compose structure) to `_detect_tools`, so idiomatic JavaScript, TypeScript,
and Docker configs are now detected with appropriate confidence instead of being
missed or mislabeled as Python.

**Tests added or updated:**
`tests/unit/test_skill_extractor.py` — added four regression tests:
`test_javascript_detected_from_keywords_without_filename` (idiomatic JS with no
filename), `test_typescript_not_misclassified_as_python` (TS syntax → TypeScript, not
Python), `test_dockerfile_directives_detected` (Dockerfile → Docker), and
`test_python_imports_not_flagged_as_javascript` (plain Python imports not mislabeled
as JS). These join the four issue-defined target tests that now pass.

**Self-review confirmation:** [x] make check passes  [x] make test-unit passes
*(In this repo "passes" = my changes introduce no new failures: `make test-unit`
went 53→49 failures — the 4 fixed targets, 0 new — and my edited files add no new
ruff/mypy errors. Full pre-existing-failure detail is in the PR's Notes for
Reviewers.)*

**Draft PR feedback received from:** none

## Week 10 — Iteration & reflection

### Reviewer feedback

**Feedback received:** [ ] Yes  [x] No — still awaiting review

**Summary of feedback:**
No reviewer feedback came in. (Per the Summer 2026 course note, reviewer feedback
isn't a feature this term.) PR #1 is open with no comments or reviews as of submission.

**How you responded:**
N/A — no feedback to respond to.

---

### Reflection

**What was harder than you expected?**
The hardest part wasn't the fix itself — it was the state of the repo. On a clean
checkout, `make test-unit` already had 53 failing tests and `make check` reported 182
lint errors, none related to my issue. That made "get the checks passing" ambiguous,
so I had to baseline everything first and redefine "done" as *introducing no new
failures*. The tooling also didn't run out of the box (no `.venv`, Docker not
running), which I hit mid-implementation.

**What did you learn about working in a large codebase?**
The bug was one line of dead code — `JS_TS_KEYWORDS` was defined but never used. The
work was *finding* that and understanding why detection was so narrow, not writing new
logic. Contributing to someone else's code means matching their conventions, keeping
the diff tightly scoped (I deliberately left an unrelated broken test alone), and
documenting pre-existing problems instead of trying to fix the whole repo.

**How did AI tools help — and where did they fall short?**
AI sped up tracing the code and drafting tests, but the scope decisions and verifying
every result against actual test runs were on me.

**What would you do differently if you started over?**
Set up the environment in Week 8 — creating the `.venv` and baselining the full
test/lint state early would have surfaced the "repo is already red" reality before I
was mid-fix, instead of discovering it in Week 9.

**What are you most proud of from this module?**
Keeping the PR honest and scoped: I documented the pre-existing failures transparently
in "Notes for Reviewers" rather than hiding them, added four regression tests that
lock the fix (including guards against the two false-positives I found), and resisted
scope creep.
