# Design

## Context

See `proposal.md` for motivation and `specs/prompt-system/spec.md` for behavioral requirements.

Resume Matcher operates in two distinct runtime modes:
1. **Coding Agent Mode (`provider: "agent"`)**: When configured in `apps/backend/data/config.json`, `GET /api/v1/status` returns `agent_mode: true`. The Next.js frontend opens `AgentPromptDialog` instead of calling backend LLM endpoints, and the user's coding agent reads `.claude/skills/resume-matcher/SKILL.md` and invokes deterministic MCP/CLI tools (`uv run --directory apps/backend resume-matcher <tool>`).
2. **Backend LLM Mode (`provider != "agent"`)**: FastAPI endpoints call `generate_resume_diffs` (`DIFF_IMPROVE_PROMPT`), `run_writing_loop` (`WRITING_RUBRIC`, `WRITING_CRITIQUE_PROMPT`, `WRITING_REWRITE_PROMPT`), `generate_cover_letter` (`config.json["cover_letter_prompt"]` or `COVER_LETTER_PROMPT`), and `generate_outreach_message` (`config.json["outreach_message_prompt"]` or `OUTREACH_MESSAGE_PROMPT`).

Both modes share the same deterministic post-processors for resumes (`apply_diffs`, `fact_violations`, `check_writing`, and `finalize_ai_resume`), while cover letters and outreach messages rely directly on prompt instructions.

## Goals / Non-Goals

**Goals:**
- Embed the actionable methodology from `Resume Knowledge File.md` (2-sentence `<30`-word summary, PAR/STAR bullet framing, strong non-blacklisted action verbs, JD terminology reframing, and qualitative scope fallback) across both Backend LLM mode and Coding Agent mode.
- Reconcile conflicts between `Resume Knowledge File.md` and Resume Matcher's deterministic validators (`writing_quality.py`, `refiner.py`, `resume_preservation.py`).
- Ensure custom feature prompts saved in `apps/backend/data/config.json` never trigger silent runtime `.format()` fallbacks in `apps/backend/app/services/cover_letter.py`.
- Preserve 100% compatibility with existing schemas, API routes, and unit/scorer test suites (`test_prompt_guardrails.py`, `test_writing_rubric_parity.py`, `test_writing_quality.py`, `test_scorers.py`).

**Non-Goals:**
- Modifying database schemas, API request/response models, or frontend React components.
- Altering `WRITING_RUBRIC` in `apps/backend/app/prompts/writing.py` (which is already locked to Challenge-Action-Result, strong action verbs, and anti-fabrication, and tested for exact byte parity with `.claude/skills/resume-matcher/SKILL.md`).

## Decisions

### 1. Reconcile Knowledge-File Rules with Deterministic Validators
- **Decision**: Override four knowledge-file instructions that conflict with factual integrity or deterministic checks:
  1. Replace *"use 'XXX' placeholders when numbers are unknown"* with explicit bans on `"XXX"` and bracketed placeholders (`[Metric]`, `[Company]`), instructing concrete qualitative scope/outcomes instead.
  2. Replace *"infer missing details based on standard industry expectations"* with strict adherence to source resume facts.
  3. Exclude *"Spearheaded"*, *"Championed"*, and *"Utilized"* from recommended action verbs because they are penalized by `AI_PHRASE_BLACKLIST` in `apps/backend/app/prompts/refinement.py` and `WRITING_RUBRIC` in `apps/backend/app/prompts/writing.py`.
  4. Omit the interactive *"3 option sets per job experience"* format in favor of the single structured output required by `ImproveDiffResult` / `ResumeData`.
- **Rationale**: In `apps/backend/app/services/writing_quality.py`, `fact_violations` rejects any rewrite that introduces a novel number (`added_number`) or an uppercase acronym token like `XXX` (`added_term:xxx`), and `finalize_ai_resume` reverts ungrounded claims.

### 2. Keep Custom Feature Prompts Free of Extra Single Braces
- **Decision**: Author `COVER_LETTER_PROMPT`, `OUTREACH_MESSAGE_PROMPT`, and the `config.json` custom prompts (`cover_letter_prompt`, `outreach_message_prompt`) with zero `{` or `}` characters outside `{job_description}`, `{resume_data}`, and `{output_language}`.
- **Alternatives Considered**: Double-escaping literal braces (`{{` / `}}`) for inline examples.
- **Rationale**: `validate_prompt_placeholders` in `apps/backend/app/prompts/__init__.py` only checks substring presence of the three required placeholders at save time, whereas `generate_cover_letter` and `generate_outreach_message` call `template.format(...)` at runtime and silently fall back to the default prompt on `KeyError` or `ValueError`. Avoiding all other braces eliminates any risk of accidental unescaped braces during user edits in the Settings UI.

### 3. Synchronize Both Backend Templates and `.claude/skills/resume-matcher/SKILL.md`
- **Decision**: Update `DIFF_IMPROVE_PROMPT`, `COVER_LETTER_PROMPT`, and `OUTREACH_MESSAGE_PROMPT` in `apps/backend/app/prompts/templates.py`, `WRITING_REWRITE_PROMPT` in `apps/backend/app/prompts/writing.py`, and `### Tailoring rules` + `## Other requests` in `.claude/skills/resume-matcher/SKILL.md`, plus persist `provider: "agent"`, `cover_letter_prompt`, and `outreach_message_prompt` in `apps/backend/data/config.json`.
- **Rationale**: In Coding Agent mode (`provider: "agent"`), the agent reads `.claude/skills/resume-matcher/SKILL.md` rather than calling backend LLM services. Updating both `.claude/skills/resume-matcher/SKILL.md` and the backend prompt constants/config ensures identical behavior whether the user runs in Coding Agent mode or switches to a backend LLM provider in Settings.

## Risks / Trade-offs

- **[Risk] Guardrail substring regressions in `test_prompt_guardrails.py` or `test_writing_rubric_parity.py`** -> **Mitigation**: Preserve every existing required substring in `DIFF_IMPROVE_PROMPT`, `COVER_LETTER_PROMPT`, and `.claude/skills/resume-matcher/SKILL.md` (`"By DEFAULT"`, `"reframe"`, `"200-300 words"`, `"200 to 300 words"`, `"Dear Hiring Manager"`, etc.), keep `WRITING_RUBRIC` untouched, and extend `test_prompt_guardrails.py` with new assertions for the added rules.
- **[Risk] `apps/backend/data/config.json` is gitignored and local-only** -> **Mitigation**: By also updating the built-in default `COVER_LETTER_PROMPT` and `OUTREACH_MESSAGE_PROMPT` in `apps/backend/app/prompts/templates.py`, fresh clones or resets automatically inherit the upgraded prompts as `cover_letter_default` and `outreach_message_default`, while `apps/backend/data/config.json` stores the active local `provider: "agent"` and custom prompt overrides.

## Migration Plan

1. Apply edits to `apps/backend/app/prompts/templates.py`, `apps/backend/app/prompts/writing.py`, `.claude/skills/resume-matcher/SKILL.md`, and `apps/backend/tests/unit/test_prompt_guardrails.py`.
2. Write `apps/backend/data/config.json` with `"provider": "agent"`, `"model": "agent"`, `"cover_letter_prompt"`, and `"outreach_message_prompt"`.
3. Run offline unit and scorer tests (`uv run --extra dev pytest tests/unit/test_prompt_guardrails.py tests/unit/test_writing_rubric_parity.py tests/unit/test_writing_quality.py tests/evals/test_scorers.py`).
4. Rollback (if needed): `git checkout -- apps/backend/app/prompts/templates.py apps/backend/app/prompts/writing.py .claude/skills/resume-matcher/SKILL.md apps/backend/tests/unit/test_prompt_guardrails.py`.

