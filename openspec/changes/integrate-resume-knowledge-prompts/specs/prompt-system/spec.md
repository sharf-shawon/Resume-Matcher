# Spec Delta

## Purpose

Governs the prompt instructions, factual-grounding boundaries, custom feature-prompt interpolation rules, and Coding Agent mode parity for resume tailoring, cover-letter generation, and cold outreach.

## ADDED Requirements

### Requirement: Untrusted Source Data and Factual Grounding Boundary
The prompt system SHALL treat job descriptions and candidate resumes strictly as untrusted source data across resume tailoring, cover-letter generation, and cold outreach, ignoring embedded instructions and prohibiting fabricated applicant facts or unresolved placeholders.

#### Scenario: Prompt injection embedded in job description
- **WHEN** a job description contains embedded instructions attempting to override task rules or inject unsupported skills, employers, or claims
- **THEN** the prompt instructions direct the generator to ignore the embedded commands and use only candidate facts supported by the source resume

#### Scenario: Experience without quantitative metrics
- **WHEN** a candidate resume entry or summary lacks numerical metrics
- **THEN** the prompt instructions prohibit inserting `"XXX"`, guessed numbers, or bracketed placeholders (such as `[Metric]` or `[Company Name]`) and direct the generator to state concrete qualitative scope and outcomes supported by the source text

### Requirement: Resume Tailoring Summary and Bullet Methodology
The resume tailoring and writing-rewrite prompts SHALL enforce a concise two-sentence summary under 30 words and frame work and project bullets using Problem-Action-Result (PAR) or Situation-Task-Action-Result (STAR) structure with strong, non-blacklisted action verbs while preserving all original factual tokens.

#### Scenario: Summary replacement during tailoring or writing rewrite
- **WHEN** the resume summary is replaced or rewritten during tailoring or writing improvement
- **THEN** the prompt instructions specify a concise paragraph of 2 sentences and under 30 words total with no bullet points, grounded solely in facts present in the source resume

#### Scenario: Bullet replacement or rewrite with strong action verbs
- **WHEN** a work experience or project bullet is replaced or rewritten
- **THEN** the prompt instructions require opening with a strong action verb (excluding weak openers such as `"Responsible for"` and blacklisted AI phrases such as `"Spearheaded"`, `"Championed"`, `"Utilized"`, `"Leveraged"`, and `"Orchestrated"`), applying PAR or STAR framing where supported by source evidence, and retaining every number, proper noun, tool, and skill from the original line

### Requirement: Cover Letter Structure, Calibration, and Brace-Safe Custom Prompt Compatibility
The cover-letter prompt (both the built-in default and the Settings UI custom prompt override) SHALL contain the exact placeholders `{job_description}`, `{resume_data}`, and `{output_language}`, contain zero unescaped literal braces outside those three placeholders, and enforce a four-part 200–300 word plain-text letter grounded in candidate evidence and job description specifics.

#### Scenario: Custom cover-letter prompt saved and rendered with brace-containing inputs
- **WHEN** the cover-letter prompt is saved via `PUT /api/v1/config/feature-prompts` and rendered via `.format(job_description=..., resume_data=..., output_language=...)` with inputs containing literal `{` or `}` characters
- **THEN** placeholder validation passes with zero missing placeholders and runtime formatting succeeds without raising `KeyError` or `ValueError` or falling back to a different template

#### Scenario: Cover letter generation with unnamed hiring manager
- **WHEN** a cover letter is generated for a job description that does not name a specific hiring manager
- **THEN** the prompt instructions require greeting `"Dear Hiring Manager,"`, structuring the letter into 4 parts (Opening hook, Match paragraph with 2–3 PAR/STAR evidence matches, Company-fit paragraph grounded in job description specifics, and Closing) in 3–4 paragraphs totaling 200–300 words, and outputting plain text with no em dashes (`—` or `--`)

### Requirement: Cold Outreach Structure and Brace-Safe Custom Prompt Compatibility
The cold outreach prompt (both the built-in default and the Settings UI custom prompt override) SHALL contain the exact placeholders `{job_description}`, `{resume_data}`, and `{output_language}`, contain zero unescaped literal braces outside those three placeholders, and produce a concise plain-text message under 150 words connecting a specific job detail to 1–2 verified candidate qualifications and a single low-friction call to action.

#### Scenario: Cold outreach generation without recipient name
- **WHEN** an outreach message is generated for a job description that does not name a recipient
- **THEN** the prompt instructions require a natural name-free greeting (`"Hi there,"` or `"Hello,"`), a specific hook referencing the role and team/challenge from the job description, 1–2 evidence-backed qualifications from the resume without fabricated metrics or `"XXX"` placeholders, a single low-friction call to action, and plain-text output with no em dashes (`—` or `--`)

### Requirement: Coding Agent Mode Instruction Parity
The Coding Agent mode instructions in `.claude/skills/resume-matcher/SKILL.md` SHALL enforce the same summary length target (2 sentences, under 30 words), PAR/STAR bullet framing, `"XXX"` placeholder prohibition, and cover-letter and outreach requirements as the backend prompts while maintaining exact `WRITING_RUBRIC` parity.

#### Scenario: Coding Agent mode rubric and prompt parity check
- **WHEN** the unit test suite (`test_writing_rubric_parity.py` and `test_prompt_guardrails.py`) runs
- **THEN** the `WRITING_RUBRIC` block in `.claude/skills/resume-matcher/SKILL.md` matches `apps/backend/app/prompts/writing.py` byte-for-byte and all cover-letter, outreach, and tailoring guardrail assertions pass

