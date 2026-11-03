# Tasks

## 1. Backend Resume Tailoring and Rewrite Prompts

- [x] 1.1 Update `DIFF_IMPROVE_PROMPT` in `apps/backend/app/prompts/templates.py` to add the untrusted-input security boundary, ban `"XXX"` and bracketed placeholders, enforce a 2-sentence `<30`-word summary target, and require PAR/STAR bullet framing with strong non-blacklisted action verbs; verify `DIFF_IMPROVE_PROMPT.format(...)` renders cleanly in Python.
- [x] 1.2 Update `WRITING_REWRITE_PROMPT` in `apps/backend/app/prompts/writing.py` to prohibit `"XXX"` and placeholder tokens and specify a 2-sentence `<30`-word summary target while keeping `WRITING_RUBRIC` untouched; verify `uv run --extra dev pytest tests/unit/test_writing_rubric_parity.py` passes.
- [x] 1.3 Add unit assertions in `apps/backend/tests/unit/test_prompt_guardrails.py` for the new `DIFF_IMPROVE_PROMPT` and `WRITING_REWRITE_PROMPT` guardrails (prompt-injection boundary, `"XXX"` prohibition, 2-sentence `<30`-word summary, and PAR/STAR rules) and verify `uv run --extra dev pytest tests/unit/test_prompt_guardrails.py` passes.

## 2. Cover Letter and Cold Outreach Prompts (Defaults and Settings Configuration)

- [x] 2.1 Update `COVER_LETTER_PROMPT` and `OUTREACH_MESSAGE_PROMPT` in `apps/backend/app/prompts/templates.py` with the verified brace-safe production templates (untrusted-source boundary, PAR/STAR match framing, `"XXX"` and placeholder prohibition, regional/tone calibration, and plain-text output contract); verify `validate_prompt_placeholders` and `string.Formatter().parse()` confirm only `{job_description}`, `{resume_data}`, and `{output_language}` are present.
- [x] 2.2 Configure `apps/backend/data/config.json` with `"provider": "agent"`, `"model": "agent"`, `"enable_cover_letter": true`, `"enable_outreach_message": true`, and the paste-ready `"cover_letter_prompt"` and `"outreach_message_prompt"` strings; verify via Python that `load_config_file()` reads the configuration and `generate_cover_letter` / `generate_outreach_message` format both overrides without fallback warnings.
- [x] 2.3 Add unit tests in `apps/backend/tests/unit/test_prompt_guardrails.py` covering `OUTREACH_MESSAGE_PROMPT` and custom prompt brace-safety and anti-placeholder invariants, and verify `uv run --extra dev pytest tests/unit/test_prompt_guardrails.py` passes.

## 3. Coding Agent Mode Instruction Parity

- [x] 3.1 Update `### Tailoring rules` and the `## Other requests` table (`Cover letter` and `Outreach message` rows) in `.claude/skills/resume-matcher/SKILL.md` to include the 2-sentence `<30`-word summary rule, PAR/STAR framing, `"XXX"` placeholder ban, and aligned cover-letter/outreach rules while preserving `"200 to 300 words"`, `"Dear Hiring Manager"`, and the `<!-- writing-rubric:start --> ... <!-- writing-rubric:end -->` block; verify `uv run --extra dev pytest tests/unit/test_writing_rubric_parity.py` passes.

## 4. End-to-End Offline Verification

- [x] 4.1 Run the full offline guardrail, writing, diff-application, preservation, and eval-scorer test suite (`cd apps/backend && uv run --extra dev pytest tests/unit/test_prompt_guardrails.py tests/unit/test_writing_rubric_parity.py tests/unit/test_writing_quality.py tests/unit/test_apply_diffs.py tests/unit/test_verify_diffs.py tests/unit/test_ai_preservation.py tests/evals/test_scorers.py -q`) and verify all tests pass with zero regressions.

