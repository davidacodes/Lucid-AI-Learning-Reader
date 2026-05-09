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
  - Deliverable: `backend/`, `frontend/`, `docs/`, `infra/`, `tests/` directories; `.specify/` containing `spec.md`, `design.md`, `tasks.md`, `constitution.md`.
  - Accept: `tree -L 2` shows the structure; specs committed.

- [ ] **T0.2** Write `constitution.md`
  - Deliverable: rules covering source-grounding, citation requirements, refusal-as-first-class, no cross-document retrieval, dyslexia-friendly defaults.
  - Accept: present in `.specify/` and reviewed.

- [ ] **T0.3** Choose stack defaults & lock versions
  - Deliverable: `pyproject.toml`, `package.json`, `docker-compose.yml` skeletons.
  - Accept: `docker compose up` brings up empty Postgres + Redis containers cleanly.

- [ ] **T0.4** CI scaffolding
  - Deliverable: GitHub Actions (or equivalent) running lint + test on push.
  - Accept: empty test suite passes on PR.

---

## Phase 1 — Ingestion Pipeline 🔒

Goal: an uploaded PDF becomes a chapter-segmented, embedded, retrievable corpus.

- [ ] **T1.1** Define core data models
  - Deliverable: SQLAlchemy models for `Document`, `Chapter`, `Chunk`; Alembic migration.
  - Accept: `alembic upgrade head` succeeds; FK relationships verified.

- [ ] **T1.2** PDF parser
  - Deliverable: function `parse_pdf(file) -> Block[]` returning text with `{page, char_start, char_end}`.
  - Accept: unit test on a 3-page sample PDF returns blocks with non-overlapping character spans and correct page numbers.

- [ ] **T1.3** EPUB parser ⚡ parallel-safe
  - Deliverable: `parse_epub(file) -> Block[]`.
  - Accept: unit test on a sample EPUB returns ordered blocks with chapter hints preserved.

- [ ] **T1.4** Chapter segmenter
  - Deliverable: `segment_chapters(blocks) -> Chapter[]` with heuristics + LLM fallback.
  - Accept: on three test documents (well-structured PDF, scanned-style PDF, EPUB), segmentation matches expected chapter list within ±1 chapter.

- [ ] **T1.5** Chunker
  - Deliverable: ~800-token recursive splitter with 150-token overlap; preserves citation metadata.
  - Accept: every chunk's `char_start/char_end` resolves back to the original text exactly.

- [ ] **T1.6** Embedding + vector store integration
  - Deliverable: pgvector setup; `embed_and_index(chunks)` function with batching and caching.
  - Accept: vector search by cosine similarity returns expected top-1 chunk for a known query.

- [ ] **T1.7** Ingestion job orchestration
  - Deliverable: Redis-backed worker that runs the full parse → segment → chunk → embed pipeline; idempotent on document hash.
  - Accept: uploading a 300-page PDF completes end-to-end in under 60 seconds on dev hardware (spec **A1**).

---

## Phase 2 — Retrieval & Generation Core 🔒

- [ ] **T2.1** Hybrid retriever
  - Deliverable: `retrieve(document_id, query, scope, k) -> Chunk[]` combining dense + BM25 with RRF.
  - Accept: on 10 hand-labeled queries, top-3 contains the gold chunk for ≥9 of them.

- [ ] **T2.2** LLM provider interface
  - Deliverable: abstract `Provider` with concrete `AnthropicProvider`; structured-output JSON mode.
  - Accept: integration test calls Claude and parses JSON response.

- [ ] **T2.3** Citation validator 🔒
  - Deliverable: `validate_citations(output, chunks)` enforcing presence + lexical overlap; returns pass/fail + reason.
  - Accept: unit tests cover (a) valid citation, (b) missing citation, (c) citation pointing to wrong chunk, (d) `NOT_IN_DOCUMENT` pass-through.

- [ ] **T2.4** Summarizer
  - Deliverable: `generate_chapter_summary(chapter)` producing 150–250 word summary with citations.
  - Accept: summaries on 5 test chapters validate cleanly and stay within length bounds.

- [ ] **T2.5** Quiz generator
  - Deliverable: `generate_quiz(chapter)` producing 5–10 questions with answers, distractors, explanations, citations.
  - Accept: every question's citation passes the validator; explanations reference the cited span.

- [ ] **T2.6** Q&A answerer
  - Deliverable: `answer_question(document_id, question)` with retrieve → generate → validate flow.
  - Accept: on the adversarial test set (≥20 out-of-document questions), refusal rate is **100%** (spec **A3**).

- [ ] **T2.7** Simplifier
  - Deliverable: `simplify(passage)` reducing reading level while preserving claims.
  - Accept: Flesch-Kincaid grade level drops ≥4 grades on 10 test passages; cited claims unchanged (spec **A5**).

- [ ] **T2.8** Glossary generator
  - Deliverable: `generate_glossary(document_id)` extracting terms with simplified definitions and citations.
  - Accept: on a sample whitepaper, ≥20 terms extracted, all citations valid.

---

## Phase 3 — SRS Scheduler

- [ ] **T3.1** FSRS implementation
  - Deliverable: `FSRSScheduler` class with `update(item, rating) -> new_state` and `due_items(user) -> Item[]`.
  - Accept: parity tests against reference FSRS implementation on 50 sample review sequences.

- [ ] **T3.2** Persist quiz history
  - Deliverable: `QuizItem` and `Review` models with FSRS state fields; migration.
  - Accept: rating an item updates `next_review` and `stability` correctly.

- [ ] **T3.3** Daily session endpoint
  - Deliverable: `GET /sessions/today` returning due items across all documents (spec **A4**).
  - Accept: items due today appear; items due tomorrow do not.

---

## Phase 4 — Application API

- [ ] **T4.1** Documents endpoints
  - Deliverable: `POST /documents`, `GET /documents/:id`, list endpoint.
  - Accept: upload → poll → `status: ready` flow works end-to-end.

- [ ] **T4.2** Summary endpoint with caching
  - Deliverable: `GET /documents/:id/chapters/:cid/summary`; first call generates, subsequent calls hit cache.
  - Accept: second call latency <200ms.

- [ ] **T4.3** Quiz endpoints
  - Deliverable: `POST` to generate/fetch, `POST /quiz-items/:id/review` for ratings.
  - Accept: full quiz lifecycle works; reviews persist and update FSRS.

- [ ] **T4.4** Q&A endpoint
  - Deliverable: `POST /documents/:id/qa` with session continuity.
  - Accept: multi-turn Q&A maintains context within a session; refusals surface cleanly.

- [ ] **T4.5** Simplify and glossary endpoints
  - Deliverable: `POST /documents/:id/simplify`, `GET /documents/:id/glossary`.
  - Accept: both return validated, cited output.

- [ ] **T4.6** Dashboard endpoint
  - Deliverable: `GET /dashboard` with retention scores, session count, per-document progress.
  - Accept: response time <2s on a corpus of 10 documents (spec **A6**).

---

## Phase 5 — Frontend

- [ ] **T5.1** Reader shell
  - Deliverable: two-pane layout, document loader, chapter navigation.
  - Accept: opening a chapter shows its text without layout shift.

- [ ] **T5.2** Dyslexia-friendly defaults
  - Deliverable: font picker (incl. OpenDyslexic), line-spacing slider, theme toggle, max-line-width control.
  - Accept: defaults meet spec **A7**; settings persist across reloads.

- [ ] **T5.3** Summary drawer
  - Deliverable: pre-chapter summary appears on chapter open; citation markers tap-through to source.
  - Accept: clicking a citation scrolls and highlights the cited span.

- [ ] **T5.4** Quiz drawer
  - Deliverable: end-of-chapter quiz UI with rating buttons (Again/Hard/Good/Easy), explanations on miss.
  - Accept: quiz state persists if user navigates away and returns.

- [ ] **T5.5** Q&A drawer
  - Deliverable: free-form question input with streaming answer + citation chips; doesn't disrupt reading position.
  - Accept: scroll position preserved when drawer opens/closes.

- [ ] **T5.6** Glossary panel
  - Deliverable: searchable term list with follow-up question affordance per term.
  - Accept: tapping a term in the reader opens the glossary entry.

- [ ] **T5.7** Dashboard view
  - Deliverable: retention chart, session calendar, per-document progress bars.
  - Accept: matches the API contract from T4.6 and renders within 2s.

---

## Phase 6 — Hallucination Hardening 🔒

This phase gates v1 release. No skipping.

- [ ] **T6.1** Adversarial test corpus
  - Deliverable: ≥20 out-of-document questions per test document, across 3 documents.
  - Accept: stored as a CI fixture.

- [ ] **T6.2** CI hallucination gate
  - Deliverable: CI job that runs the corpus and fails the build on any non-refusal.
  - Accept: gate is wired into the default branch protection.

- [ ] **T6.3** Citation-verification audit
  - Deliverable: sampling job that re-validates 1% of generated citations daily in production.
  - Accept: alert fires on validation failure rate >0.1%.

---

## Phase 7 — Polish & Release

- [ ] **T7.1** Observability: structured logs, request tracing, LLM call metering.
- [ ] **T7.2** Cost dashboard: per-document token spend, embedding spend.
- [ ] **T7.3** Local-first deployment doc: Docker Compose quickstart.
- [ ] **T7.4** Backup/restore for user data (quiz history especially).
- [ ] **T7.5** Onboarding flow: upload → first chapter summary in under 90 seconds.

---

## Cross-Cutting Acceptance — v1 Release Gate

All must be true before tagging v1:

- [ ] All Phase 1–6 tasks checked off.
- [ ] Spec criteria **A1–A7** verified by automated tests.
- [ ] Adversarial hallucination corpus: 100% refusal rate.
- [ ] Manual UX pass on three real documents (an AI book chapter, a research paper, a vendor whitepaper).
- [ ] `constitution.md` rules audited against shipped behavior.
