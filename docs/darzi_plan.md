# Darzi — Delivery Plan (v0.1)

> Companion to `spec.md`. Epics map to the spec's phased roadmap (§11). Phase 1
> stories are fully fleshed out; later phases are epic-level stubs to expand once
> Phase 1 lands. Each story is sized to finish in a single focused session.
>
> Suggested issue labels: `epic`, `story`, `phase-1`…`phase-5`, `core`, `infra`,
> `intake`, `generate`, `deliver`, `security`.

---

## Epics

| ID | Epic | Phase | Goal |
|----|------|-------|------|
| **E1** | MVP core (no sending) | 1 | Pasted URL → tailored résumé + cover letter, local output |
| **E2** | Voice & retrieval quality | 2 | Output is convincingly "in my voice"; right case studies surface |
| **E3** | Email channel + approval gate | 3 | Can send, only after Telegram approval; replies tracked |
| **E4** | Source intake & triage | 4 | Multi-source discovery + fit-score auto-triage + scheduling |
| **E5** | ATS apply + web dashboard | 5 | Form-based applies; read-only management UI |

Cross-cutting: **security** (egress allowlist, secret proxy, action gates) is
threaded through E3+ rather than a standalone epic — see spec §8.

---

## E1 — MVP core (Phase 1) — detailed stories

> Definition of done for the epic: from a clean checkout + `data/` seeded, I can
> run one command with a job URL and get a tailored résumé + cover letter written
> to disk, with the honesty guard active. No network sending of any kind.

### S1.1 — Project scaffold
- `pyproject.toml` (Python 3.12+, `uv`), package `darzi/`, `ruff` + `pytest`
  configured, `.env.example`, `README` with quickstart.
- **AC:** `uv sync` works; `ruff check` and `pytest` run green on an empty suite.

### S1.2 — Postgres + pgvector + migrations
- Migration tooling (alembic or plain SQL); create the MVP subset of the spec §4
  schema: `profile_variants`, `case_studies` (with `embedding vector`), `jobs`,
  `applications` (minimal), `audit_log`.
- **AC:** `make db-up` (Docker Postgres+pgvector) + migrate creates all tables;
  a smoke test inserts/queries a vector.

### S1.3 — Profile & case-study ingestion
- Loader that reads `data/` (résumé variants + case studies as md/txt), parses
  into the schema, and computes embeddings for `case_studies`.
- **AC:** running the loader populates `profile_variants` and `case_studies`;
  re-running is idempotent (upsert, no dupes).

### S1.4 — MCP server skeleton + hard gates
- stdio MCP server registering all spec §3.1 tools as stubs; implement the
  **guardrail layer** now even if tools are stubs: approval-token check on
  `send_*` (returns refusal), audit-log wrapper on all tools.
- **AC:** server starts; an MCP client lists tools; calling `send_application`
  without a token returns a structured refusal; every call writes an audit row.

### S1.5 — `fetch_url` tool
- HTTP fetch (httpx) → readability/markdown cleanup → `{markdown, meta}`.
  Handles redirects; returns a clear error on block/non-HTML.
- **AC:** given a real public job URL, returns clean markdown; given a 403,
  returns a typed error, not an exception.

### S1.6 — `extract_jobs` tool
- LLM extraction from markdown → validated `Job` pydantic model(s)
  (title, company, location, comp, description, link).
- **AC:** extraction on 3 sample postings yields correct structured fields;
  malformed input degrades gracefully.

### S1.7 — `score_fit` tool
- Score a `Job` against the profile → `{score 0–1, reasons[], gaps[]}`.
- **AC:** an obviously-relevant role scores high, an irrelevant one low;
  reasons reference real profile signals.

### S1.8 — `retrieve_case_studies` tool
- pgvector top-k over `case_studies` for the job's requirements, re-ranked by
  tag/skill overlap; returns top 3–6.
- **AC:** for a given JD, returns the most relevant case studies ahead of
  irrelevant ones; respects `k`.

### S1.9 — `tailor_resume` tool + honesty guard
- Select base variant + retrieved case studies → tailored résumé text.
- **Honesty guard (code):** post-generation check rejects/flags any
  employer/title/date/metric not present in the source fact set.
- **AC:** output reorders/rephrases real content; an injected fabrication is
  caught by the guard and the call fails with a clear reason.

### S1.10 — `write_cover_letter` tool
- Cover letter grounded in the job + profile, plain voice (full voice guide is
  E2). Same honesty guard applies.
- **AC:** produces a coherent, job-specific letter; no fabricated claims pass
  the guard.

### S1.11 — End-to-end CLI flow
- `darzi apply-dry-run <url>` orchestrates fetch → extract → score → retrieve →
  tailor → cover-letter, writing artifacts to `output/` and a `jobs` row.
- **AC:** one command on a real URL produces résumé + cover letter files + a DB
  record; runs offline-of-sending (no outbound apply).

### S1.12 — Tests + CI
- Unit tests for the guard, extraction parsing, retrieval ranking; GitHub Actions
  running `ruff` + `pytest` on PRs.
- **AC:** CI green on main; honesty-guard has explicit pass/fail test cases.

**Phase-1 build order:** S1.1 → S1.2 → (S1.3 ∥ S1.4) → S1.5 → S1.6 →
(S1.7 ∥ S1.8) → S1.9 → S1.10 → S1.11 → S1.12.

---

## E2 — Voice & retrieval quality (Phase 2) — stub
- Derive `voice_guide` from `voice_samples`; inject into generation.
- Tune retrieval (hybrid vector + tag) until featured case studies are right.
- Eval harness: A/B sample outputs, track "sounds like me" rating.
- Depends on: E1.

## E3 — Email channel + approval gate (Phase 3) — stub
- Telegram approval card (summary + fit + artifacts) → Approve/Edit/Skip → token.
- `send_application` via dedicated Gmail; Gmail reply ingestion → `applications`.
- Security: egress allowlist, secret proxy, spend caps, identity hygiene (spec §8).
- Depends on: E1; harness decision (OpenFang vs Hermes) informs wiring.

## E4 — Source intake & triage (Phase 4) — stub
- Tier-1 ZipRecruiter API adapter; tier-2 Greenhouse/Lever/Ashby fetch +
  `resolve_ats`; tier-3 headful browser (anti-bot playbook, CAPTCHA→Telegram).
- Fit-score auto-triage; scheduled discovery runs.
- Depends on: E1, E3 (for review/notify of discovered jobs).

## E5 — ATS apply + web dashboard (Phase 5) — stub
- Form-fill for Greenhouse/Lever/Ashby (field taxonomy + answers_cache).
- Read-mostly web dashboard over Postgres (pipeline, tracker, fit tuning).
- Depends on: E3, E4.

---

## How to use this with web containers
- One story = one issue = one focused session/branch/PR.
- Each session has `docs/spec.md` (the what/why) + this plan + the issue's AC.
- Flesh out E2–E5 stories into issues only when their phase begins; keep the
  backlog shallow until then.
