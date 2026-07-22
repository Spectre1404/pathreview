# Solution plan

**Issue:** #148 — Skill extractor fails to detect JavaScript and TypeScript

## Understand

`SkillExtractor._detect_languages` in `ingestion/parsers/skill_extractor.py`
detects JS/TS only from a `.js`/`.ts` filename, an `import`/`require` statement, or
the literal string `package.json`. Idiomatic code (`const`/`let`/arrow functions/
`function`/`export`) is never flagged. The class defines a `JS_TS_KEYWORDS` set
(line 30) for exactly these tokens, but `_detect_languages` never references it —
dead code. This is the core defect.

Reproduction (Week 8, in `JOURNAL.md`) confirmed three concrete mechanisms:

1. **Dead constant.** `JS_TS_KEYWORDS` is unused, so keyword signals never fire.
2. **`require(...)` slips the one narrow signal.** `re.search(r"\b(import|require)\s+")`
   (line 179) requires whitespace after `require`, but `require('fs')` has `(` —
   no match. So idiomatic JS detects nothing.
3. **TS is mislabeled as Python.** The Python annotation regex
   `:\s*(int|str|float|bool|list|dict)` (line 160) matches `id: string` because
   `string` starts with `str`, scoring the TS sample as Python (0.70).

The two Docker target tests fail for a related reason: `_detect_tools` (line 267)
substring-matches the literal `"docker"`, which a real Dockerfile/compose file never
contains — so a Dockerfile is misread as Python (via `requirements.txt`, line 162)
and a compose file detects nothing.

**Root cause:** JS/TS keyword + TS-syntax signals are never wired into
`_detect_languages`; Docker structural signals (Dockerfile directives, compose YAML)
are never wired into `_detect_tools`.

**Scope note:** #148's title is JS/TS-specific, but it lists four target tests as the
"done" criteria — two of which (`test_devops_tool_detection`,
`test_docker_compose_detection`) are Docker detection. The fix therefore spans
`_detect_languages` **and** `_detect_tools`.

## Map

Files I expect to touch:

- `ingestion/parsers/skill_extractor.py`
  - `_detect_languages` (lines ~143–214): wire in JS keyword/syntax signals; add
    TS-vs-JS disambiguation; fix the `require(` matcher; tighten the Python
    annotation regex so `string` no longer matches `str`.
  - `_detect_tools` (lines ~263–276): add Docker detection from Dockerfile
    directives and docker-compose YAML structure (not just the literal `"docker"`).
- `tests/unit/test_skill_extractor.py`: **read-only** — the four target tests already
  define "done." I will not weaken them. (Possible separate one-line fix for the
  pre-existing typo on line 138 — see Risks, needs approval.)

Not touched: `agent/tools/skill_extractor.py` (different `BaseTool` class),
`api/`, `core/`, schema, frontend. The in-scope file is imported only by its test.

## Plan

1. Confirm baseline: `PYTHONPATH=. python -m pytest tests/unit/test_skill_extractor.py -q`
   → 13 pass, 4 in-scope fail. (Done.)
2. **JavaScript signal.** In `_detect_languages`, add a whole-word match against
   JS-*distinctive* tokens (`const`, `let`, `=>`, `require`, `console.`,
   `function `, `export`) — deliberately excluding tokens shared with Python
   (`import`/`class`/`async`) so Python-only text does not gain a false JavaScript
   skill. Fix the `require` check to match `require(` as well.
3. **TypeScript disambiguation.** Detect TS-specific syntax (`interface `, `enum `,
   `type X =`, typed params `: string`/`: number`, generics like `Promise<...>`,
   access modifiers). If any TS signal is present, label the result `TypeScript`;
   otherwise `JavaScript`. Preserve the existing filename-based `.ts` → TypeScript.
4. **Stop the TS→Python misfire.** Add a trailing `\b` to the Python annotation
   regex (`:\s*(int|str|float|bool|list|dict)\b`) so `string` no longer matches
   `str`. Verify `test_text_with_python_imports` and
   `test_text_with_python_type_annotations` still pass (they also detect Python via
   `def`/`import`, so this should be safe).
5. **Docker signal.** In `_detect_tools`, add Docker when Dockerfile directives are
   present (`FROM`, `RUN`, `EXPOSE`, `CMD`, `ENTRYPOINT`, `COPY`, `WORKDIR`) or when
   compose structure is present (`services:` plus `version:`/`image:`/`build:`/
   `ports:`). Keep the existing literal-`docker` match.
6. Re-run the target file: 4 in-scope tests green, and the 13 currently-passing tests
   stay green (no regressions).
7. Run `make check` + `make test-unit` once `make setup` has created `.venv`
   (currently blocked — see Risks). Format/lint/type clean before the PR.

## Inputs & outputs

**Method I'm changing:** `_detect_languages(text, filename, skills_dict) -> None`
(mutates `skills_dict`) and `_detect_tools(text, skills_dict) -> None`. Public entry
`extract_skills(text, filename=None) -> list[SkillDetection]` is unchanged.

Target behaviors (from the four tests, no code fences / filenames):

| Input | Expected skill present |
|-------|------------------------|
| `const fs = require('fs'); console.log(...)` | JavaScript |
| `export interface User {...} async getUser(id: string): Promise<User>` | TypeScript |
| `FROM python:3.9 / RUN pip install / EXPOSE 8000` | Docker |
| `version: / services: / ports:` | Docker |

Invariants the other tests enforce and I must keep: every result is a
`SkillDetection` with `0.0 <= confidence <= 1.0` (`test_confidence_scores_are_floats`,
`test_mixed_language_text`), Python still detected from imports/annotations, React and
frameworks unaffected. New confidences will follow the existing
`min(0.95, 0.6 + n*0.1)` evidence-count pattern (values proposed, tuned during
implementation).

## Risks & unknowns

1. **Shared-keyword false positives.** `import`/`class`/`async` appear in both Python
   and JS. If the JS signal keys on them, Python-only samples wrongly gain a
   JavaScript skill. Mitigation: base JS detection on JS-distinctive tokens only
   (step 2). Verify pure-Python tests gain no JavaScript entry.
2. **Tightening the Python regex could shift Python detection.** Adding `\b` is
   intended to only stop `string`→`str`. Must re-run the two Python tests to confirm
   no regression before finalizing.
3. **Local verification is partial.** `make test-unit`/`make check` can't run — the
   full suite fails collection on missing deps (`structlog`, etc.) because `.venv`
   isn't set up. I verify via the single target file until `make setup` succeeds
   (needs Docker Desktop running). Unknown whether `make setup` completes cleanly on
   this machine — I'll attempt it before the Week 9 PR.
4. **Docker scope beyond the title.** Steps 5 touches DevOps detection, wider than
   "JS/TS." Justified because #148 lists both Docker tests as done-criteria; noted so
   the PR description is honest about scope.
5. **Pre-existing test typo (out of scope).** `test_database_technology_detection`
   line 138 reads `skill_names = [s.name for s in skill_names]` (uses the variable
   before it's defined) — an existing bug unrelated to #148. Decision needed: leave
   it, or fix the one-liner as a courtesy. **Will not touch without approval.**

## Edge cases

- **Python text containing `import`/`class`/`async`** → must NOT be labeled
  JavaScript (drives the distinctive-token design).
- **Mixed-language text** (`test_mixed_language_text`) → multiple languages detected,
  all with valid confidences; JS via `require`, TS via `interface`, Python via
  `async def`.
- **`.jsx`/`.tsx`** → already handled by `_detect_react`; my change must not
  double-count or break React detection.
- **Minimal snippet (one JS token)** → detected, confidence capped at 0.95.
- **Dockerfile that also mentions Python** (`FROM python:3.9`) → Docker present;
  Python may also appear (tests only require Docker present).
- **Compose file without the literal `docker`** → detected via `services:` structure.
