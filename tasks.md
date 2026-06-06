# Draft
# Tasks: Lucid AI Learning Reader

Execution checklist derived from `spec.md` and `design.md`. Tasks are grouped into phases that build on each other; each task has an ID, a deliverable, and acceptance criteria. Check items off as completed.

**Conventions**
- `[ ]` = not started, `[~]` = in progress, `[x]` = done
- Tasks marked **🔒 gated** must pass their acceptance check before downstream tasks begin
- Tasks marked **⚡ parallel-safe** can run alongside the prior task

---

## Phase 0 — Project Foundations

- [ ] **T0.1** Initialize repo structure
  - Deliverable: `backend/`, `bot/`, `docs/`, `infra/`, `tests/` directories; `.specify/` containing `spec.md`, `design.md`, `tasks.md`, `constitution.md`.
  - Accept: `tree -L 2` shows the structure; specs committed.

- [ ] **T0.2** Write `constitution.md`
  - Deliverable: rules covering source-grounding, citation requirements, refusal-as-first-class, no cross-document retrieval, document-scoped answers only.
  - Accept: present in `.specify/` and reviewed.

- [ ] **T0.3** Choose stack defaults & lock versions
  - Deliverable: `pyproject.toml`, `docker-compose.yml` skeletons; `discord.py` v2, `openai` SDK, and `pydantic-ai` pinned.
  - Accept: `docker compose up` brings up empty Postgres + Redis containers cleanly.

- [ ] **T0.4** CI scaffolding
  - Deliverable: GitHub Actions running lint + test on push.
  - Accept: empty test suite passes on PR.

---

## Phase 1 — Ingestion Pipeline 🔒

Goal: an uploaded PDF becomes a chapter-segmented, embedded, retrievable corpus.

- [ ] **T1.1** Define core data models
  - Deliverable: SQLAlchemy models for `Document` (incl. `discord_user_id`), `Chapter`, `Chunk`; Alembic migration.
  - Accept: `alembic upgrade head` succeeds; FK relationships verified.

- [ ] **T1.2** PDF parser
  - Deliverable: function `parse_pdf(file) -> Block[]` returning text with `{page, char_start, char_end}`.
  - Accept: unit test on a 3-page sample PDF returns blocks with non-overlapping character spans and correct page numbers.

- [ ] **T1.3** Chapter segmenter
  - Deliverable: `segment_chapters(blocks) -> Chapter[]` using TOC detection, font-size jumps, and numbered heading heuristics; LLM fallback for unstructured docs.
  - Accept: on three test PDFs (well-structured, scanned-style, minimal formatting), segmentation matches expected chapter list within ±1 chapter.

- [ ] **T1.4** Chunker
  - Deliverable: ~800-token recursive splitter with 150-token overlap; preserves citation metadata.
  - Accept: every chunk's `char_start/char_end` resolves back to the original text exactly.

- [ ] **T1.5** Embedding + vector store integration
  - Deliverable: pgvector setup; `embed_and_index(chunks)` using OpenAI `text-embedding-3-small` with batching and hash-based caching.
  - Accept: vector search by cosine similarity returns expected top-1 chunk for a known query; re-upload of same file hash skips re-embedding.

- [ ] **T1.6** Ingestion job orchestration
  - Deliverable: Redis-backed worker running the full parse → segment → chunk → embed pipeline; idempotent on document hash; notifies Discord user on completion.
  - Accept: uploading a 300-page PDF completes end-to-end in under 60 seconds on dev hardware (spec **A1**).

---

## Phase 2 — Retrieval & Generation Core 🔒

- [ ] **T2.1** Hybrid retriever
  - Deliverable: `retrieve(document_id, query, scope, k) -> Chunk[]` combining dense + BM25 with reciprocal rank fusion.
  - Accept: on 10 hand-labeled queries, top-3 contains the gold chunk for ≥9 of them.

- [ ] **T2.2** Pydantic AI agents + output models
  - Deliverable: three Pydantic AI agents (`summarizer_agent`, `quizzer_agent`, `answerer_agent`) each with a typed output model (`SummaryOutput`, `QuizOutput`, `AnswerOutput | Refusal`); all wired to OpenAI GPT-4o with `retries=1`.
  - Accept: integration tests call each agent and confirm (a) structured output parses correctly, (b) a deliberately malformed model response triggers a retry, (c) a `Refusal` is returned when the prompt signals out-of-document.

- [ ] **T2.3** Citation lexical verifier 🔒
  - Deliverable: `verify_citations(output, chunks)` — runs *after* Pydantic AI validates schema; checks each cited chunk ID exists and that key claims have ≥0.6 token-overlap with the cited text; returns pass/fail + reason.
  - Accept: unit tests cover (a) valid citation with sufficient overlap, (b) citation pointing to wrong chunk, (c) overlap below threshold, (d) `Refusal` input is a no-op pass-through.

- [ ] **T2.4** Summarizer
  - Deliverable: `generate_chapter_summary(chapter)` using `summarizer_agent`; produces 150–250 word `SummaryOutput` with citations; runs lexical verifier before returning.
  - Accept: summaries on 5 test chapters pass schema validation, pass citation verification, and stay within length bounds.

- [ ] **T2.5** Quiz generator
  - Deliverable: `generate_quiz(chapter)` using `quizzer_agent`; produces 5–10 MCQ `QuizOutput` questions with answers, distractors, explanations, citations; runs lexical verifier before returning.
  - Accept: every question's citation passes the verifier; explanations reference the cited span.

- [ ] **T2.6** Q&A answerer
  - Deliverable: `answer_question(document_id, question)` using `answerer_agent`; retrieve → generate → verify flow; returns `AnswerOutput` with citations or `Refusal`.
  - Accept: on the adversarial test set (≥20 out-of-document questions), refusal rate is **100%** (spec **A3**).

- [ ] **T2.7** Simplifier
  - Deliverable: `simplify(passage)` reducing reading level while preserving factual claims.
  - Accept: Flesch-Kincaid grade level drops ≥4 grades on 10 test passages; cited claims unchanged (spec **A5**).

- [ ] **T2.8** Glossary generator
  - Deliverable: `generate_glossary(document_id)` extracting terms with simplified definitions and citations.
  - Accept: on a sample whitepaper, ≥20 terms extracted, all citations valid.

---

## Phase 3 — SRS Scheduler

- [ ] **T3.1** SM-2 implementation
  - Deliverable: `SM2Scheduler` class with `update(item, rating: int) -> new_state` (rating 0–5; ratings < 3 reset interval) and `due_items(discord_user_id) -> Item[]`.
  - Accept: unit tests verify interval progression and ease-factor updates against the SM-2 specification on 20 sample review sequences.

- [ ] **T3.2** Persist quiz history
  - Deliverable: `QuizItem` and `Review` models with SM-2 state fields (`ease_factor`, `interval`, `repetitions`, `next_review`, `last_reviewed`); Alembic migration.
  - Accept: rating an item updates `next_review` and `ease_factor` correctly; raw `Review` row is appended.

- [ ] **T3.3** Daily session endpoint
  - Deliverable: `GET /sessions/today` returning due items across all of a user's documents (spec **A4**).
  - Accept: items where `next_review <= now()` appear; items due tomorrow do not.

---

## Phase 4 — Application API

- [ ] **T4.1** Documents endpoints
  - Deliverable: `POST /documents`, `GET /documents/:id`, list endpoint scoped to `discord_user_id`.
  - Accept: upload → poll → `status: ready` flow works end-to-end.

- [ ] **T4.2** Summary endpoint with caching
  - Deliverable: `GET /documents/:id/chapters/:cid/summary`; first call generates, subsequent calls hit cache.
  - Accept: second call latency <200ms.

- [ ] **T4.3** Quiz endpoints
  - Deliverable: `POST` to generate/fetch quiz, `POST /quiz-items/:id/review` for SM-2 ratings.
  - Accept: full quiz lifecycle works; reviews persist and correctly update SM-2 state.

- [ ] **T4.4** Q&A endpoint
  - Deliverable: `POST /documents/:id/qa` with session continuity.
  - Accept: multi-turn Q&A maintains context within a session; refusals surface cleanly.

- [ ] **T4.5** Simplify and glossary endpoints
  - Deliverable: `POST /documents/:id/simplify`, `GET /documents/:id/glossary`.
  - Accept: both return validated, cited output.

- [ ] **T4.6** Dashboard endpoint
  - Deliverable: `GET /dashboard` with retention scores, session count, per-document progress scoped to `discord_user_id`.
  - Accept: response time <2s on a corpus of 10 documents (spec **A6**).

---

## Phase 5 — Discord Bot

- [ ] **T5.1** Bot scaffolding
  - Deliverable: `discord.py` v2 app with slash command registration, bot token config, and health-check startup log.
  - Accept: bot comes online, slash commands appear in a test server, and `bot.is_ready()` is true.

- [ ] **T5.2** `/upload` command
  - Deliverable: accepts a PDF attachment, forwards file to `POST /documents`, replies with an ingestion-progress message, edits it to a success embed on `document.ready`.
  - Accept: uploading a real PDF produces a "ready" confirmation in Discord; duplicate upload is a no-op with a user-friendly message.

- [ ] **T5.3** Active-document state
  - Deliverable: per-user in-memory (or Redis) store of `active_document_id`; commands default to it when no document argument is supplied.
  - Accept: after `/upload`, subsequent commands work without specifying the document; user can override with an explicit argument.

- [ ] **T5.4** `/read` command
  - Deliverable: posts chapter summary as a Discord embed; offers a **Next Chapter** button to advance.
  - Accept: summary text and page citation render correctly in the embed; button advances chapter ordinal.

- [ ] **T5.5** `/quiz` command + MCQ interactions ⚡ parallel-safe
  - Deliverable: sends questions one at a time as embeds with A/B/C/D button components; after answer, disables buttons and appends explanation + citation blockquote.
  - Accept: correct answer highlights correctly; wrong answer shows explanation; quiz state survives a bot restart (persisted via `Quiz`/`QuizItem` models).

- [ ] **T5.6** `/review` command ⚡ parallel-safe
  - Deliverable: fetches SM-2 due items via `GET /sessions/today` and runs the same MCQ button flow as `/quiz`.
  - Accept: only items with `next_review <= now()` appear; SM-2 state updates after each rating.

- [ ] **T5.7** `/ask` command
  - Deliverable: takes a free-form question string, calls `POST /documents/:id/qa`, posts answer embed with citation blockquotes; spills to a thread if over Discord's 4096-char embed limit.
  - Accept: out-of-document questions receive the "not in document" message; answers include page citations.

- [ ] **T5.8** `/simplify`, `/glossary`, `/dashboard` commands ⚡ parallel-safe
  - Deliverable: `/simplify` accepts a quoted passage; `/glossary` accepts a term; `/dashboard` posts a retention embed.
  - Accept: all three commands return cited, validated output matching the API contract from T4.5–T4.6.

- [ ] **T5.9** Rate-limit queue
  - Deliverable: outbound Discord message queue that respects per-channel and global rate limits; backs off on 429 responses.
  - Accept: a quiz flow with 10 rapid answers does not produce a Discord API error.

---

## Phase 6 — Hallucination Hardening 🔒

This phase gates v1 release. No skipping.

- [ ] **T6.1** Adversarial test corpus
  - Deliverable: ≥20 out-of-document questions per test document, across 3 documents.
  - Accept: stored as a CI fixture.

- [ ] **T6.2** CI hallucination gate
  - Deliverable: CI job that runs the corpus and fails the build on any non-refusal.
  - Accept: gate is wired into default branch protection.

- [ ] **T6.3** Citation-verification audit
  - Deliverable: sampling job that re-validates 1% of generated citations daily in production.
  - Accept: alert fires on validation failure rate >0.1%.

---

## Phase 7 — Polish & Release

- [ ] **T7.1** Observability: structured logs, request tracing, LLM call metering.
- [ ] **T7.2** Cost dashboard: per-document token spend, embedding spend.
- [ ] **T7.3** Local-first deployment doc: Docker Compose quickstart with bot token setup.
- [ ] **T7.4** Backup/restore for user data (quiz history especially).
- [ ] **T7.5** Onboarding flow: upload → first chapter summary in Discord in under 90 seconds.

---

## Cross-Cutting Acceptance — v1 Release Gate

All must be true before tagging v1:

- [ ] All Phase 1–6 tasks checked off.
- [ ] Spec criteria **A1–A6** verified by automated tests.
- [ ] Adversarial hallucination corpus: 100% refusal rate.
- [ ] Manual UX pass on three real documents (an AI book chapter, a research paper, a vendor whitepaper) via Discord.
- [ ] `constitution.md` rules audited against shipped behavior.
