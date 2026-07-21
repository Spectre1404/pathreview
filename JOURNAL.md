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
