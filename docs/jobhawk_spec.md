# JobHawk — Design Spec (v0.1, DRAFT)

> An agentic job-discovery + tailored-application system, built as a portable
> MCP core on top of an existing "claw"-family agent harness.
>
> Status: brainstorming draft. Nothing here is locked except where noted.

---

## 1. Goal & philosophy

Automate the **discovery → tailoring → drafting → (reviewed) submission** of job
applications, with materials written in the user's voice and grounded only in the
user's real experience.

Design principles:

1. **Generation is the value; auto-submit is the risk.** Invest in tailoring +
   voice; gate every outbound action behind human review until trust is earned.
2. **Portable core, swappable harness.** Domain logic lives in a standalone MCP
   server so the agent harness can change without a rewrite.
3. **Enforce in code, not in prompts.** Safety/honesty constraints are hard rules
   in the MCP server, not polite instructions in a skill (the model can ignore
   those).
4. **Cleanest door per source.** API > plain fetch > headful browser > manual.
5. **Honest tailoring.** Reorder, re-emphasize, rephrase real content — never
   invent experience, metrics, titles, or dates.

### Non-goals (for now)
- LinkedIn automation (ToS/ban risk; user doesn't need it).
- High-volume "spray and pray" applying (hurts quality, deliverability, reputation).
- Fully autonomous submission with no human in the loop (at least through Phase 3).

---

## 2. Architecture overview

```
┌─ HARNESS (a "claw") — not built; chosen later ────────────────┐
│  channels (Telegram primary) · agent loop · cron/scheduler ·  │
│  sandboxing · LLM routing · conversational memory             │
├─ OFF-THE-SHELF SKILLS — reused from harness/ecosystem ────────┤
│  Gmail/email · Google Docs/Sheets · headful browser control   │
├─ JOBHAWK SKILL (SKILL.md / HAND.toml) — built (the brain) ────┤
│  workflow: discover → extract → score → tailor → draft →      │
│            queue-for-review → (approved) send → track         │
│  voice rules · per-source policy · human-in-loop gates        │
├─ JOBHAWK MCP SERVER — built (the muscle, PORTABLE CORE) ──────┤
│  deterministic tools + hard-coded guardrails                  │
│        └── Postgres + pgvector (profile, jobs, applications)  │
└────────────────────────────────────────────────────────────────┘
```

**Harness decision: PARKED.** Leading candidates (security-first, MCP-capable):
OpenFang (scheduled-worker "Hands" model, prompt-injection scanner, WASM sandbox)
and Hermes (mature, self-authored skills = no supply chain, persistent memory).
OpenClaw is deprioritized for this use case due to 2026 security incidents
(CVE-2026-25253 one-click RCE, ClawHavoc supply-chain skills, prompt-injection
attacks in the wild). The MCP core keeps this reversible.

---

## 3. The portable core: JobHawk MCP server

Language: Python, stdio MCP server. Single source of domain truth + the hard gates.

### 3.1 Tool surface (draft)

```
# Discovery & intake
search_source(source, filters)        -> [JobStub]      # API/feed sources
fetch_url(url)                          -> {markdown, meta}
extract_jobs(content)                   -> [Job]         # LLM extraction
resolve_ats(job)                        -> Job           # follow Indeed->Greenhouse/Lever

# Triage
score_fit(job)                          -> {score, reasons, gaps}

# Knowledge / retrieval
retrieve_case_studies(job, k)           -> [CaseStudy]   # pgvector + tag filter
get_profile_field(category, field)      -> value

# Generation
tailor_resume(job, base_variant)        -> {doc_id, pdf_path}
write_cover_letter(job)                 -> {doc_id, text}
answer_application_question(q, type, opts) -> answer      # with cache

# Delivery (GATED)
queue_for_review(application)            -> review_id     # -> Telegram card
approve_review(review_id, edits?)        -> token
send_application(review_id)              -> result        # REFUSES w/o valid approval token
send_email / send_sms                   -> result        # REFUSES w/o gate

# Tracking
log_application(...) / update_status(...) / get_status(filter) -> ...
```

### 3.2 Hard-coded guardrails (enforced here, not in the skill)
- `send_application`, `send_email`, `send_sms` require a valid, unexpired approval
  token tied to a specific `review_id`. No token → refuse.
- `tailor_resume` / `write_cover_letter` operate over a fact set pulled from the
  profile store; a post-generation check rejects output containing
  employers/titles/dates/metrics not present in the source facts.
- Per-source rate limits + daily caps enforced server-side.
- All outbound actions append to an audit log.

---

## 4. Data model (Postgres + pgvector)

Single store; pgvector for the only thing that needs similarity (case-study
retrieval). Everything else is relational.

```
profile_variants(id, label, target_role, positioning, full_text)
case_studies(id, title, role, domain, skills[], metrics, summary,
             full_text, url, embedding vector)
voice_samples(id, kind, text)            -- for tone extraction
voice_guide(id, derived_style_spec)      -- generated once from voice_samples

jobs(id, source, url, ats_url, raw, title, company, location,
     comp, description, fit_score, fit_reasons, status, first_seen, last_seen)
applications(id, job_id, resume_doc, cover_letter_doc, channel,
             review_id, approved_at, sent_at, status, reply_text, reply_at)
answers_cache(id, question_norm, type, answer)   -- reuse across apps

sources(id, name, tier, method, api_key_ref, rate_limit, daily_cap)
audit_log(id, ts, action, target, payload_redacted)
```

Retrieval logic: embed job requirements → pgvector top-k over `case_studies`,
re-rank by tag/skill overlap → feed the top 3–6 into `tailor_resume`.

---

## 5. Source intake strategy (tiered)

| Tier | Sources | Method |
|---|---|---|
| 1 — Official API | ZipRecruiter Publisher Search API; feeds | Authenticated REST, no browser |
| 2 — Plain fetch | Greenhouse (`boards-api.greenhouse.io`), Lever (`api.lever.co`), Ashby, RSS | Simple HTTP JSON |
| 3 — Headful residential browser | Indeed, Glassdoor, JS/anti-bot sites | Real Chrome/Edge, residential IP, human cadence, CAPTCHA→human |
| 4 — Off-limits / manual | LinkedIn; hard-CAPTCHA/C&D sites | Don't automate |

Key tactic: **discover broadly, then resolve to the ATS.** Most Indeed/ZipRecruiter
listings point at a Greenhouse/Lever page with a clean JSON endpoint — fetch the
authoritative posting there (tier 2) instead of scraping the aggregator.

### Anti-bot playbook (tier 3)
- Real persistent browser profile (never the user's default); stable fingerprint.
- Human pacing: randomized delays, realistic dwell/scroll, occasional back-nav.
- Low volume, jittered schedule, single concurrency.
- CAPTCHA → pause + Telegram handoff; never retry-storm; back off on first block.
- Use real installed browser (real TLS/JS fingerprint); hide automation flags.
- Expectation: best-effort, degrade gracefully. Rely on tiers 1–2 for volume.

---

## 6. Application channels

| Channel | Method | Auth needed | Priority |
|---|---|---|---|
| Email apply | Dedicated Gmail (API) | no (just the agent's mailbox) | Phase 3 — first |
| ATS web form (Greenhouse/Lever/Ashby) | Headful browser fill, or API where available | usually no | Phase 5 |
| Workday-style portals | Headful browser | varies | Deferred |
| Job-board native apply | Mostly avoided | — | Skip |

**Approval gate (all channels):** agent drafts → Telegram card (job summary, fit
score, tailored resume + cover letter) → user taps Approve / Edit / Skip →
`send_application` executes only with the resulting token. Auto-send threshold can
be raised per-channel once trust is established (e.g. email-apply, fit > 0.85).

---

## 7. Voice & tailoring

- Capture: ingest resume variants + writing samples → derive a reusable
  `voice_guide` (tone, vocabulary, sentence rhythm, what to avoid).
- Tailor: select relevant case studies → reorder/re-emphasize a base resume
  variant for the JD; write a cover letter in the voice guide.
- Honesty guard (code): reject any generated artifact referencing facts not in
  the profile source set. Tailoring may rephrase; it may not invent.
- Output to Google Docs (formatted) + PDF for attachment.

---

## 8. Security model

Threat model: agent ingests attacker-authored content (job posts) AND holds
real-world capabilities (send email as the agent identity, Twilio, PII corpus).
Primary risks: prompt-injection → capability abuse / exfiltration; credential
theft; PII leak; host/LAN compromise.

Controls (priority order):
1. **Egress allowlist** — outbound restricted to LLM API, Gmail, Twilio, target
   job sites. Blocks exfiltration/abuse even on full compromise. Highest ROI.
2. **Secrets out of model context** — auth proxy injects keys at the network
   boundary (exe.dev-style, or self-hosted proxy + firewall). Agent never holds
   raw keys.
3. **Action gates + caps** — code-enforced approval tokens; hard LLM/Twilio spend
   caps; Twilio geo/premium-rate disabled.
4. **Identity hygiene** — dedicated Google account used for nothing else; minimal
   OAuth scopes; not a recovery address for anything.
5. **Ingest/action separation** — parse untrusted job HTML in an isolated context;
   use authenticated/trusted actions only on known-good URLs.
6. **Scoped, rotatable creds** + a 5-minute rotate/revoke/rebuild runbook.

Note: the auth proxy protects the *secret*, not the *action* — capability abuse
(e.g. "send 1000 SMS") is still possible, so controls 1, 3, 5 remain essential.

---

## 9. Deployment topology (current direction)

- **Host:** dedicated personal laptop, dedicated non-admin Windows user profile
  (no personal data on it). Residential IP — the key enabler for tier-3 anti-bot.
- **Runtime:** WSL2 for the Linux dev/runtime convenience (NOT treated as a
  security boundary). Postgres + MCP server in WSL2; headful Chrome/Edge on
  Windows driven via CDP/remote-debug.
- **Isolation:** network egress allowlist (router or host firewall); optional
  upgrade to a real VM (Hyper-V) for a true boundary; optional VLAN/guest-network
  segmentation from personal devices.
- **Comms:** Twilio for SMS/calls (Google Voice has no API). Dedicated Gmail/
  Workspace identity for email-apply + reply tracking.

Alternative considered: cloud VPS (exe.dev KVM VM + auth proxy) — better secret
isolation, but datacenter IP fails tier-3 anti-bot. Hybrid possible later.

---

## 10. Interfaces

- **CLI** — power user, batch runs, debugging.
- **Telegram** — primary daily driver: approvals, "apply to this URL", status.
- **Web dashboard** — last; read-mostly view of the Postgres data (pipeline,
  tracker, fit-score tuning).

---

## 11. Phased roadmap

1. **MVP (no sending):** Postgres schema + load profile/case studies →
   `fetch → extract → score → tailor → cover letter` for one pasted URL → Google
   Doc. Validate quality.
2. **Voice + retrieval:** voice guide + case-study matching until output is "you."
3. **Email channel + Telegram approval gate:** can send, only with a tap.
4. **Source intake + triage:** ZipRecruiter API + Greenhouse/Lever fetch +
   Indeed headful; fit-score auto-triage; scheduling.
5. **ATS form apply + web dashboard.**

---

## 12. Open questions
- Harness: OpenFang vs Hermes — decide after MVP (core is portable).
- How much tier-3 (Indeed) volume is actually worth the maintenance vs leaning on
  tiers 1–2?
- Auto-send trust threshold per channel — start fully manual; tune later.
- Where does the voice_guide live — MCP/Postgres vs harness memory?
- Resume rendering pipeline (reuse AIHawk's reportlab approach vs Google Docs export).
```
