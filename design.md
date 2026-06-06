# Draft
# Design: Lucid AI Learning Reader

This document describes *how* the system in `spec.md` is built. It is intentionally implementation-leaning but stops short of file-level decisions, which live in `tasks.md`.

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Discord Client                           │
│   User uploads PDF • slash commands • button interactions       │
└─────────────────────────┬───────────────────────────────────────┘
                          │  Discord Gateway / Webhook
┌─────────────────────────▼───────────────────────────────────────┐
│                        Discord Bot                              │
│   Command router • Event handler • Message formatter            │
└─────────────────────────┬───────────────────────────────────────┘
                          │  internal calls
┌─────────────────────────▼───────────────────────────────────────┐
│                       Application API                           │
│   Documents • Chapters • Summaries • Quizzes • Reviews • Q&A    │
└──┬──────────────┬───────────────┬───────────────┬───────────────┘
   │              │               │               │
┌──▼────────┐ ┌───▼──────┐ ┌──────▼──────┐ ┌──────▼──────────────┐
│ Ingestion │ │ Retrieval│ │ Generation  │ │  SRS Scheduler      │
│ Pipeline  │ │ (vector) │ │  (LLM)      │ │  (SM-2)             │
└──┬────────┘ └───┬──────┘ └──────┬──────┘ └──────┬──────────────┘
   │              │               │               │
┌──▼──────────────▼───────────────▼───────────────▼──────────────┐
│       Storage: Postgres (relational) • Vector DB • Blob        │
└────────────────────────────────────────────────────────────────┘
```

Five subsystems: **Ingestion**, **Retrieval**, **Generation**, **SRS**, and **Application API**, fronted by a **Discord Bot** layer.

## 2. Subsystems

### 2.1 Ingestion Pipeline

**Job:** Turn an uploaded PDF into a structured, searchable, chapter-segmented corpus.

Stages, run as a queued job:

1. **Parse.** PDF → text via `pdfplumber` (preserves page numbers and char offsets). Output: ordered list of `Block` records with `{page, char_start, char_end, text}`.
2. **Segment chapters.** Heuristic stack: detect TOC entries, font-size jumps, and numbered heading patterns in PDF. Falls back to LLM-assisted segmentation for unstructured documents. Output: `Chapter[]` with title, ordinal, and block range.
3. **Chunk.** Within each chapter, recursive character splitter at ~800 tokens with ~150 overlap. Each chunk preserves citation metadata: `{document_id, chapter_id, page_start, page_end, char_start, char_end}`.
4. **Embed.** Batch through OpenAI `text-embedding-3-small`; persist vectors plus chunk metadata.
5. **Index.** Write to vector store; emit `document.ready` event; notify the user via Discord message.

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
4. **Refusal pass-through.** If the model emits `NOT_IN_DOCUMENT`, the bot surfaces a clear "this isn't in the document" Discord message — no fallback to general knowledge.

### 2.4 SRS Scheduler

**Job:** Decide which quiz items to resurface and when.

- Algorithm: **SM-2** (SuperMemo 2) — simple, well-understood, and sufficient for v1.
- Each `QuizItem` has: `ease_factor`, `interval`, `repetitions`, `last_reviewed`, `next_review`.
- On quiz answer: update SM-2 state with the rating (0–5 scale; ratings < 3 reset the interval).
- On session start: query items where `next_review <= now()` and prepend them to the session, capped at N per session.

### 2.5 Application API

**Job:** Expose the above to the Discord bot over a stable internal interface.

Selected endpoints:

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/documents` | Upload; returns `document_id`, kicks off ingestion. |
| `GET` | `/documents/:id` | Metadata + chapter list. |
| `GET` | `/documents/:id/chapters/:cid/summary` | Pre-reading summary (cached). |
| `POST` | `/documents/:id/chapters/:cid/quiz` | Generate or fetch quiz. |
| `POST` | `/quiz-items/:id/review` | Submit rating; updates SM-2 state. |
| `GET` | `/sessions/today` | Items due for review across all docs. |
| `POST` | `/documents/:id/qa` | Free-form Q&A. |
| `POST` | `/documents/:id/simplify` | Simplify a passage. |
| `GET` | `/documents/:id/glossary` | Auto-generated glossary. |
| `GET` | `/dashboard` | Aggregate retention + progress. |

## 3. Data Model

```
Document(id, title, source_hash, format, uploaded_at, status, discord_user_id)
Chapter(id, document_id, ordinal, title, char_start, char_end)
Chunk(id, chapter_id, page_start, page_end, char_start, char_end, text, embedding_ref)
Summary(id, chapter_id, body, citations[], generated_at, model_version)
Quiz(id, chapter_id, generated_at)
QuizItem(id, quiz_id, kind, prompt, correct_answer, distractors[], explanation,
        citation_chunk_id, citation_span,
        ease_factor, interval, repetitions, next_review, last_reviewed)
Review(id, quiz_item_id, rating, reviewed_at)
GlossaryEntry(id, document_id, term, definition, citations[])
QASession(id, document_id, started_at)
QAMessage(id, session_id, role, content, citations[])
```

Vector store holds chunk embeddings keyed by `chunk.id`. Blob store holds the original uploaded file.

## 4. Discord Bot UX

The bot is the only user-facing surface. All interaction happens via slash commands, Discord message components (buttons, select menus), and embeds.

**Core slash commands:**

| Command | Behaviour |
|---|---|
| `/upload` | User attaches a PDF; bot begins ingestion and replies with a progress message, then a confirmation embed when ready. |
| `/read [document] [chapter]` | Bot posts the chapter summary as an embed, then offers a **Start Reading** button. |
| `/quiz [document] [chapter]` | Bot sends quiz questions one at a time as embeds with A/B/C/D buttons; after each answer shows explanation + citation. |
| `/review` | Surfaces SM-2 due items from all documents in an interactive quiz flow. |
| `/ask [question]` | Free-form Q&A against the active document; bot replies with answer embed including cited page numbers. |
| `/simplify` | User pastes or quotes a passage; bot replies with a simplified version + citation. |
| `/glossary [term]` | Bot looks up the term in the auto-glossary and replies with definition + source page. |
| `/dashboard` | Bot posts an embed showing retention scores, session count, and per-document progress. |

**Interaction patterns:**

- Quiz answers are collected via Discord **button components** (A / B / C / D). After submission the buttons are disabled and the correct answer + explanation embed is appended.
- Long content (summaries, answers) uses Discord embeds with field truncation; oversized text spills into a thread.
- Citations are rendered as `> "quoted text" — p. 42` blockquotes inside the embed, not as hyperlinks.
- The bot maintains a per-user "active document" state so most commands work without specifying a document each time.

## 5. Hallucination Prevention — Detail

This is the load-bearing guarantee of the product, so it gets its own section.

- **Inputs to the model are sandboxed.** Source chunks are wrapped in `<source id="...">` tags. The system prompt forbids using anything outside those tags as factual basis.
- **Output schema is strict JSON** with required `citations` array. No citations → response rejected and regenerated.
- **Citations are verified** by checking that the answer's key claims have ≥0.6 token-overlap (or pass an NLI entailment check) against the cited chunk text. Failures trigger regeneration once, then refusal.
- **Refusal is a first-class output**, not a fallback. The model is primed via system prompt and few-shots to emit `NOT_IN_DOCUMENT` when grounding is absent.
- **Adversarial test set** of out-of-document questions runs in CI. Any regression below 100% refusal rate fails the build.

## 6. Stack

- **Backend:** Python 3.12, FastAPI, Postgres, pgvector (or Qdrant), Redis for job queue.
- **Discord bot:** `discord.py` (v2) with slash command and component support.
- **Ingestion:** `pdfplumber`, `tiktoken`, custom chapter segmenter.
- **LLM:** OpenAI GPT-4o for generation; `text-embedding-3-small` for embeddings.
- **Embeddings:** Cached aggressively; re-upload of the same file hash skips re-embedding.
- **Auth:** Single Discord user ID scoped per deployment; no login flow required.
- **Deployment:** Docker Compose for local; single-host target for self-hosting.

## 7. Open Design Questions

- **Vector store:** pgvector keeps the stack simpler; Qdrant scales further. v1 leans pgvector unless document counts grow large.
- **Local-LLM mode:** Defer to v2 (per spec decision).
- **Rate limiting:** Discord has per-user and global rate limits; the bot needs a queue to avoid hitting them during quiz flows with many rapid interactions.
