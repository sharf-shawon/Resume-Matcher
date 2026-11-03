# Proposal

## Why

Resume Matcher supports two execution paths—Coding Agent mode (`provider: "agent"`, driven by `.claude/skills/resume-matcher/SKILL.md` and the `resume-matcher` CLI) and Backend LLM mode (driven by `apps/backend/app/prompts/templates.py`, `apps/backend/app/prompts/writing.py`, and Settings UI custom feature prompts in `apps/backend/data/config.json`)—but neither path currently integrates the user's resume-writing knowledge methodology (`Resume Knowledge File.md`: 2-sentence `<30`-word summaries, explicit PAR/STAR bullet framing with non-blacklisted strong action verbs, explicit prompt-injection boundaries, and strict prohibition of `"XXX"` or bracketed placeholders). Aligning both the backend prompt templates, Settings UI custom prompts, and Coding Agent instructions ensures identical factual integrity and writing standards across all modes.

## What Changes

- **Refine Backend Resume Tailoring Prompts (`apps/backend/app/prompts/templates.py` & `apps/backend/app/prompts/writing.py`)**:
  - Add an explicit prompt-injection security boundary to `DIFF_IMPROVE_PROMPT`.
  - Enforce the knowledge-file default summary target (2 sentences, under 30 words, no bullet points) in `DIFF_IMPROVE_PROMPT` and `WRITING_REWRITE_PROMPT`.
  - Instruct PAR (Problem-Action-Result) / STAR (Situation-Task-Action-Result) bullet framing using strong non-blacklisted action verbs while preserving all original numbers, proper nouns, tools, and skills.
  - Explicitly forbid `"XXX"` and bracketed placeholders (`[metric]`, `[tool]`) when quantitative metrics are absent from the source resume, falling back to concrete qualitative scope and outcomes.
- **Upgrade Cover Letter & Cold Outreach Prompts (`apps/backend/app/prompts/templates.py` & Settings UI `config.json`)**:
  - Update `COVER_LETTER_PROMPT` and `OUTREACH_MESSAGE_PROMPT` with untrusted-input boundaries, PAR/STAR match framing, strict anti-fabrication rules, `"XXX"` / placeholder bans, and brace-safe `{job_description}`, `{resume_data}`, and `{output_language}` interpolation.
  - Provide paste-ready custom prompts for `cover_letter_prompt` and `outreach_message_prompt` in `apps/backend/data/config.json` alongside `provider: "agent"`.
- **Sync Coding Agent Mode Instructions (`.claude/skills/resume-matcher/SKILL.md`)**:
  - Update `### Tailoring rules` and `## Other requests` (`Cover letter` and `Outreach message`) in `.claude/skills/resume-matcher/SKILL.md` so Coding Agent mode applies the exact same 2-sentence `<30`-word summary target, PAR/STAR framing, `"XXX"` ban, and cover-letter/outreach structure while preserving the `<!-- writing-rubric:start --> ... <!-- writing-rubric:end -->` block and `test_writing_rubric_parity.py` invariants.
- **Lock Guardrail Invariants in Unit Tests (`apps/backend/tests/unit/test_prompt_guardrails.py`)**:
  - Add deterministic unit assertions verifying the new anti-placeholder, summary-length, PAR/STAR, and prompt-injection rules across tailoring, cover-letter, and outreach prompts.

## Capabilities

### New Capabilities
- `prompt-system`: Grounded prompt templates, custom feature-prompt configuration, and Coding Agent mode instructions for resume tailoring, cover-letter generation, and cold outreach.

### Modified Capabilities

## Impact

- **Backend Prompt Templates**: `apps/backend/app/prompts/templates.py`, `apps/backend/app/prompts/writing.py`
- **Coding Agent Skill**: `.claude/skills/resume-matcher/SKILL.md`
- **Local Configuration (Gitignored)**: `apps/backend/data/config.json` (`provider: "agent"`, `cover_letter_prompt`, `outreach_message_prompt`)
- **Unit Test Suite**: `apps/backend/tests/unit/test_prompt_guardrails.py`, `apps/backend/tests/unit/test_writing_rubric_parity.py`

