# Draft
# Specification: Lucid AI Learning Reader

> *Read it. Understand it. Remember it.*

## 1. Overview

A personal, AI-powered reading companion that turns dense technical material — AI books, research papers, whitepapers — into a guided learning experience. The product is dyslexia-first: every chapter opens with a plain-English summary that primes understanding before reading, and closes with a quiz that reinforces retention. Unlike a generic chatbot, the system is architecturally constrained to answer **only** from the uploaded document. If a paper describes v2.3.1 of a tool, the quiz reflects v2.3.1 — never the model's general knowledge of the tool.

## 2. Problem Statement

Technical readers — especially neurodivergent ones — face three compounding problems:

1. **Comprehension friction.** Dense academic prose, jargon, and assumed background knowledge make it hard to start a chapter with confidence.
2. **Retention decay.** Without active recall, most of what's read is forgotten within a week.
3. **Hallucination risk in AI tools.** Generic Q&A chatbots blend the document with the model's training data, so a reader studying a specific paper can't trust that the answer reflects *that* paper.

Existing tools solve at most one of these. None solve all three with a strict source-grounding guarantee.

## 3. Goals

- Lower the activation cost of starting a chapter through pre-reading summaries.
- Drive long-term retention through spaced-repetition quizzing on auto-generated questions.
- Make whitepaper-grade language accessible at an undergrad reading level.
- Guarantee every AI-generated artifact (summary, question, glossary entry, answer) is traceable to specific spans of the source document.

## 4. Non-Goals

- **Not** a general-purpose chatbot. The system refuses questions outside the uploaded document.
- **Not** a multi-user collaborative platform. Single-user, personal-use scope.
- **Not** a document editor or annotation tool with shared markup.
- **Not** a replacement for the source text — it is a scaffolding layer around it.
- **No** content recommendation engine; the user brings their own documents.

## 5. Target User

Primary: a self-directed learner working through technical AI literature (papers, books, whitepapers) who benefits from structured pre-reading framing, active recall, and dyslexia-friendly presentation. Comfortable uploading PDFs and EPUBs; values trust in sources over breadth of features.

## 6. Core Features (User-Facing)

| # | Feature | What it does |
|---|---|---|
| F1 | **Chapter pre-summary** | On opening a chapter, shows a plain-English summary framing the content before the user reads. |
| F2 | **End-of-chapter quiz** | Generates 5–10 questions per chapter; explains incorrect answers with citation back to the source. |
| F3 | **Spaced repetition** | Persists quiz history across sessions; resurfaces weak items on an SM-2-style schedule. |
| F4 | **Whitepaper simplifier** | Rewrites PhD-level passages to undergrad reading level on demand. |
| F5 | **Auto-glossary** | Extracts technical terms and offers simplified definitions; user can ask follow-ups on any term. |
| F6 | **Free-form Q&A** | At any point, user can ask questions about the loaded document without losing reading position. |
| F7 | **Hallucination prevention** | Every generated artifact carries citations to source spans; ungrounded answers are refused. |
| F8 | **Learning dashboard** | Shows retention scores, session history, and per-document progress. |

## 7. User Stories

- **As a reader**, I upload a PDF and the system segments it into chapters so I can move through it linearly.
- **As a reader starting a chapter**, I see a 150–250 word plain-English summary so I know what to expect before diving in.
- **As a reader finishing a chapter**, I take a 5–10 question quiz and, when I get one wrong, I see the correct answer with a quote from the source.
- **As a returning reader**, weak items from prior sessions appear at the start of today's session.
- **As a reader hitting jargon**, I tap a term and get an undergrad-level definition grounded in how *this document* uses it.
- **As a reader confused mid-paragraph**, I ask a free-form question and get an answer with citations, without losing my scroll position.
- **As a reader tracking progress**, I open a dashboard and see retention per document and per topic.

## 8. Acceptance Criteria

The system is considered to meet spec when:

- **A1.** Uploading a PDF or EPUB produces a navigable chapter structure within 60 seconds for a 300-page document.
- **A2.** Every quiz question, summary, and glossary entry includes a citation (page + character span) to the source.
- **A3.** When asked a question whose answer is not in the document, the system explicitly says so rather than fabricating one. This is verified by an adversarial test set (≥20 out-of-document questions, 100% refusal rate required).
- **A4.** Quiz history persists across sessions and the SRS scheduler correctly surfaces items whose review interval has elapsed.
- **A5.** The simplifier reduces Flesch-Kincaid grade level by at least 4 grades on selected passages while preserving factual claims (verified by spot-check of citations).
- **A6.** The dashboard renders retention scores, session count, and document progress within 2 seconds.

## 9. Constraints

- **Source-grounding is non-negotiable.** No feature may generate content that the system cannot cite back to the document.
- **Single-user, local-first where feasible.** User data (quiz history, documents) stays on the user's device or a single user-owned backend.
- **Cost-aware.** Embedding and generation calls must be batched and cached; re-opening a document must not re-embed it.
- **Privacy.** Uploaded documents are not used to train any model and are not shared with third parties beyond the chosen LLM provider's standard inference path.

## 10. Decisions (formerly Open Questions)

- **LLM provider:** OpenAI GPT-4o for generation and embeddings (v1).
- **Offline mode:** Stretch goal / v2 — v1 requires internet connectivity.
- **Platform:** Discord bot (v1). The reading companion is delivered as a Discord bot; PDF files are uploaded directly in Discord and all interaction happens in-channel or via DMs.
- **SRS algorithm:** Vanilla SM-2.
- **File-format scope:** PDF only for v1.

## 11. Success Metrics

- Quiz completion rate per chapter ≥70% over a four-week usage window.
- Self-reported comprehension lift (pre/post survey) on a sample of chapters.
- Hallucination rate (measured against the adversarial test set) of 0%.
- Median time-to-first-summary after upload ≤60 seconds.
