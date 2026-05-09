# Draft
# Design: Lucid AI Learning Reader

This document describes *how* the system in `spec.md` is built. It is intentionally implementation-leaning but stops short of file-level decisions, which live in `tasks.md`.

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         Client (Web UI)                         │
│   Reader pane • Quiz pane • Dashboard • Glossary • Q&A drawer   │
└─────────────────────────┬───────────────────────────────────────┘
                          │  HTTPS / JSON
┌─────────────────────────▼───────────────────────────────────────┐
│                       Application API                           │
│   Documents • Chapters • Summaries • Quizzes • Reviews • Q&A    │
└──┬──────────────┬───────────────┬───────────────┬───────────────┘
   │              │               │               │
┌──▼────────┐ ┌───▼──────┐ ┌──────▼──────┐ ┌──────▼──────────────┐
│ Ingestion │ │ Retrieval│ │ Generation  │ │  SRS Scheduler      │
│ Pipeline  │ │ (vector) │ │  (LLM)      │ │  (SM-2 / FSRS)      │
└──┬────────┘ └───┬──────┘ └──────┬──────┘ └──────┬──────────────┘
   │              │               │               │
┌──▼──────────────▼───────────────▼───────────────▼──────────────┐
│       Storage: Postgres (relational) • Vector DB • Blob       │
└────────────────────────────────────────────────────────────────┘
```

Five subsystems: **Ingestion**, **Retrieval**, **Generation**, **SRS**, and **Application API**. Each has a narrow contract.

## 2. Subsystems

### 2.1 Ingestion Pipeline

**Job:** Turn an uploaded document into a structured, searchable, chapter-segmented corpus.

Stages, run as a queued job:

1. **Parse.** PDF → text via `pdfplumber` (preserves page numbers and char offsets); EPUB → text via `ebooklib`. Output: ordered list of `Block` records with `{page, char_start, char_end, text}`.
2. **Segment chapters.** Heuristic stack: detect TOC entries, `<h1>`/`<h2>` in EPUB, font-size jumps and numbered patterns in PDF. Falls back to LLM-assisted segmentation for unstructured documents. Output: `Chapter[]` with title, ordinal, and block range.
3. **Chunk.** Within each chapter, recursive character splitter at ~800 tokens with ~150 overlap. Each chunk preserves citation metadata: `{document_id, chapter_id, page_start, page_end, char_start, char_end}`.
4. **Embed.** Batch through the embedding model; persist vectors plus chunk metadata.
5. **Index.** Write to vector store; emit `document.ready` event.

Re-uploading the same document hash is a no-op.

### 2.2 Retrieval Layer

**Job:** Given a query and a scope (whole document, chapter, or term), return the top-k chunks with citations.

- Hybrid retrieval: dense (cosine over embeddings) + sparse (BM25) with reciprocal rank fusion.
- All retrieval is **scoped to a single document**. Cross-document retrieval is not supported.
- Returns: `Chunk[]` with text, citation metadata, and score.
- Used by every generation call.

### 2.3 Generation Layer

**Job:** Call the LLM with strict source-grounding.

Three patterns, each with a fixed prompt template:

- **Summarizer** (`generate_chapter_summary`): input = chapter chunks, output = 150–250 word plain-English summary with inline citations.
- **Quizzer** (`generate_quiz`): input = chapter chunks, output = 5–10 questions, each with answer, distractors (for MCQ), explanation, and citation span.
- **Answerer** (`answer_question`): input = retrieved chunks + user question, output = answer with citations OR explicit "not in document" refusal.

**Hallucination prevention — multi-layer:**

1. **Prompt-level.** System prompt instructs the model to answer only from `<source>` blocks and to emit a fixed `NOT_IN_DOCUMENT` token when the answer isn't there.
2. **Citation enforcement.** Every output requires a citation pointing to a real chunk ID. Outputs without citations are rejected at the validator.
3. **Citation verification.** Validator checks that the cited span actually contains substring evidence for the claim (lexical overlap threshold + optional NLI check).
4. **Refusal pass-through.** If the model emits `NOT_IN_DOCUMENT`, the API surfaces a clear "this isn't in the document" message to the user — no fallback to general knowledge.

### 2.4 SRS Scheduler

**Job:** Decide which quiz items to resurface and when.

- Algorithm: **FSRS** (Free Spaced Repetition Scheduler) — better than SM-2 for sparse review patterns, well-documented, open-source.
- Each `QuizItem` has: ease, interval, last_reviewed, next_review, stability, difficulty.
- On quiz answer: update FSRS state with the rating (Again / Hard / Good / Easy).
- On session start: query items where `next_review <= now()` and prepend them to the session, capped at N per session.

### 2.5 Application API

**Job:** Expose the above to the client over a stable REST/JSON interface.

Selected endpoints:

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/documents` | Upload; returns `document_id`, kicks off ingestion. |
| `GET` | `/documents/:id` | Metadata + chapter list. |
| `GET` | `/documents/:id/chapters/:cid/summary` | Pre-reading summary (cached). |
| `POST` | `/documents/:id/chapters/:cid/quiz` | Generate or fetch quiz. |
| `POST` | `/quiz-items/:id/review` | Submit rating; updates FSRS state. |
| `GET` | `/sessions/today` | Items due for review across all docs. |
| `POST` | `/documents/:id/qa` | Free-form Q&A. |
| `POST` | `/documents/:id/simplify` | Simplify a passage. |
| `GET` | `/documents/:id/glossary` | Auto-generated glossary. |
| `GET` | `/dashboard` | Aggregate retention + progress. |

## 3. Data Model

```
Document(id, title, source_hash, format, uploaded_at, status)
Chapter(id, document_id, ordinal, title, char_start, char_end)
Chunk(id, chapter_id, page_start, page_end, char_start, char_end, text, embedding_ref)
Summary(id, chapter_id, body, citations[], generated_at, model_version)
Quiz(id, chapter_id, generated_at)
QuizItem(id, quiz_id, kind, prompt, correct_answer, distractors[], explanation,
        citation_chunk_id, citation_span,
        fsrs_stability, fsrs_difficulty, next_review, last_reviewed)
Review(id, quiz_item_id, rating, reviewed_at)
GlossaryEntry(id, document_id, term, definition, citations[])
QASession(id, document_id, started_at)
QAMessage(id, session_id, role, content, citations[])
```

Vector store holds chunk embeddings keyed by `chunk.id`. Blob store holds the original uploaded file.

## 4. Frontend

A two-pane reader: source text on the left (configurable font, spacing, theme — OpenDyslexic available), a contextual drawer on the right that swaps between Summary, Quiz, Glossary, and Q&A. Reading position is preserved in client storage so the drawer never disrupts scroll.

State management: lightweight (Zustand or similar). Server state via React Query for caching summaries, quizzes, and glossary. Citations render as tappable footnote-style markers that scroll the reader pane to the cited span and highlight it.

## 5. Hallucination Prevention — Detail

This is the load-bearing guarantee of the product, so it gets its own section.

- **Inputs to the model are sandboxed.** Source chunks are wrapped in `<source id="...">` tags. The system prompt forbids using anything outside those tags as factual basis.
- **Output schema is strict JSON** with required `citations` array. No citations → response rejected and regenerated.
- **Citations are verified** by checking that the answer's key claims have ≥0.6 token-overlap (or pass an NLI entailment check) against the cited chunk text. Failures trigger regeneration once, then refusal.
- **Refusal is a first-class output**, not a fallback. The model is trained-in (via prompt and few-shots) to emit `NOT_IN_DOCUMENT` when grounding is absent.
- **Adversarial test set** of out-of-document questions runs in CI. Any regression below 100% refusal rate fails the build.

## 6. Stack

- **Backend:** Python 3.12, FastAPI, Postgres, pgvector (or Qdrant), Redis for job queue.
- **Ingestion:** `pdfplumber`, `ebooklib`, `tiktoken`, custom segmenter.
- **LLM:** Anthropic Claude (default), pluggable provider interface for future local models.
- **Embeddings:** Provider's hosted embedding model; cached aggressively.
- **Frontend:** React + TypeScript, Vite, Tailwind, React Query, Zustand.
- **Auth:** Single-user; long-lived token in v1.
- **Deployment:** Docker Compose for local; single container target for self-hosting.

## 7. Open Design Questions

- **Vector store:** pgvector keeps the stack simpler; Qdrant scales further. v1 leans pgvector unless document counts grow large.
- **Local-LLM mode:** Ollama integration is straightforward at the provider interface but adds prompt-tuning surface area. Defer to v2.
- **Mobile:** Reader UX on small screens needs its own design pass; v1 is desktop-first.
