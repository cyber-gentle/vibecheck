# VibeCheck AI — Architecture

**Status:** DRAFT, pending review by Engineer A. Items marked **TBD** are still open.

## 1. Purpose

VibeCheck AI is a terminal application that imports festival reviews, asks an AI model to classify each one, verifies the model's output in Python, computes statistics deterministically, and answers questions using only the supplied reviews.

## 2. Module map and owners

```text
vibecheck/
├── main.py              (shared)  entry point
├── app/
│   ├── __init__.py
│   ├── models.py        A   shape of review / classification records
│   ├── ingest.py        A   import_reviews: read CSV/JSON, validate, reject duplicates
│   ├── provider.py      A   HTTPS client, timeout, bounded retries, no secrets logged
│   ├── classify.py      A   classify_review: prompt, strict parse, quote check
│   ├── storage.py       A   save/load JSON files
│   ├── cache.py         A   hash-based cache
│   ├── report.py        B   counts, percentages, topics, coverage
│   ├── retrieve.py      B   keyword/token-overlap search, top 5
│   ├── qa.py            B   grounded Q&A, cited-ID validation, abstention
│   ├── evaluation.py    B   gold-set comparison, accuracy, error analysis
│   └── cli.py           B   menu 0-8, re-prompting, export
├── data/
│   ├── synthetic_reviews.csv
│   └── gold_set.json
├── tests/
├── docs/
├── requirements.txt
├── README.md
├── .env.example
└── .gitignore
```

## 3. Data flow and trust boundaries

```text
reviews CSV/JSON
   │ import_reviews (A)        UNTRUSTED input -> validated
   ▼
validated reviews
   │ classify_review (A)       review text sent to AI as quoted DATA
   ▼
AI reply                       UNTRUSTED output -> must pass validation
   │ strict parser (A)
   ▼
verified classifications ──► cache (A)
   │
   ├─► report.py (B)           Python only, verified records only
   ├─► retrieve.py + qa.py (B) <=5 reviews sent to AI, cited IDs checked
   ├─► evaluation.py (B)       gold set vs model labels
   └─► cli.py / export (B)
```

### Rules

- Review text and model output are untrusted until validated.
- Reports read only verified classifications.
- The model never computes totals or percentages.

## 4. Data contracts

### 4.1 Review record

`id` (unique), `event` (non-empty), `text` (5–500 chars), `created_at` (`YYYY-MM-DD`).

### 4.2 Classification record

| Field | Rule |
| :--- | :--- |
| `review_id` | must match the input ID |
| `sentiment` | `positive`, `negative`, `neutral` or `mixed` |
| `topics` | a list, 0–3 unique values from: `sound`, `food`, `queue`, `staff`, `pricing`, `safety`, `venue`, `other` |
| `evidence_quote` | non-empty exact substring of the review text |
| `short_reason` | 160 characters or fewer |

Metadata stored with each verified record: provider/model ID, prompt/schema version, source-text SHA-256, optional timestamp. **TBD:** exact field names, and whether metadata is nested or flat.

### 4.3 Failed / pending analysis

Never store a false sentiment for an unanalysed review.

- **Statuses:** `pending`, `analyzed`, `analysis_failed`.
- **Failure record fields:** `review_id`, `status`, `reason`, `attempts`.

### 4.4 Q&A response

`answer` and `cited_review_ids`. Every cited ID must belong to the retrieved subset. If evidence is insufficient, the answer is exactly: `Insufficient evidence in the supplied reviews.`

## 5. Storage

Separate JSON files for: original reviews, verified classifications, errors/status, model metadata, and Q&A session summaries.

The classifications file is a dictionary keyed by `review_id`.

**TBD:** file names and folder.

## 6. Provider interface

The provider is a function that takes a prompt and returns raw text. The caller (`classify.py`, `qa.py`) parses and validates the reply. Tests use a small fake function that returns a fixed string.

## 7. AI-derived vs Python-computed

| AI-derived (inference, label as such) | Computed by Python |
| :--- | :--- |
| sentiment, topics, evidence quote, short reason | counts, percentages, topic frequency |
| Q&A answer text | retrieval, citation validation, coverage (analyzed N / total N) |

## 8. Decisions log

### Analysis state

- **Options:** three statuses / two statuses (pending = no record) / three plus stale
- **Choice:** three statuses (`pending`, `analyzed`, `analysis_failed`)
- **Why:** "not yet tried" and "tried and failed" are different states and the CLI must show both. Staleness is detected by comparing the source hash, so it needs no status of its own.

### Failure record contents

- **Options:** id + status / plus reason / plus reason and attempts
- **Choice:** `review_id`, `status`, `reason`, `attempts`
- **Why:** reports need status for counts; the CLI needs reason and attempts to show failures and retries transparently.

### Classification storage layout

- **Options:** list / dictionary keyed by `review_id` / one combined file
- **Choice:** dictionary keyed by `review_id`
- **Why:** direct lookup, easy cache checks, no duplicate IDs by construction. A combined file would mix source reviews with AI analysis, which the brief says to keep separate.

### Provider interface

- **Options:** function passed in / object with a method / provider returns a parsed dictionary
- **Choice:** function taking a prompt and returning raw text
- **Why:** matches the brief's `call_model`, keeps validation on the caller's side so cited IDs can be checked independently, and makes a fake provider trivial.

### Empty topics

- **Options:** `[]` / `["other"]` / allow both
- **Choice:** `[]` means "no identifiable topic"
- **Why:** `other` stays for real topics outside the list. The report shows how many reviews had no topic so topic percentages are not misleading.

### Retrieval tokenisation

- **Options:** lowercase + strip punctuation + split / plus stop words / plus stemming
- **Choice:** lowercase, strip punctuation, split on spaces, remove stop words
- **Why:** ignores filler words without the extra logic of a hand-written stemmer. The stop-word list is short and kept in the repo so tests and reviewers can see it.

### Retrieval tie-breaking

- **Options:** score then `review_id` / score then `created_at` / score then text length
- **Choice:** score descending, then `review_id` ascending
- **Why:** IDs are unique, so no ties remain; the rule is easy to state and to prove deterministic.

### Percentage display

- **Options:** one decimal / whole numbers / count beside percentage
- **Choice:** count beside the percentage, one decimal place (e.g. `7 of 20, 35.0%`)
- **Why:** shows the sample size next to each percentage, and one decimal makes hand-checking easy.

### Zero denominator

- **Options:** N/A with counts / 0.0% / refuse to build the report
- **Choice:** N/A for percentages, with pending and failed counts still shown
- **Why:** nothing is invented, and the user still sees what is missing.

### File ownership: `storage.py` and `evaluation.py`

- **Options:** A / B / shared
- **Choice:** `storage.py` to A, `evaluation.py` to B
- **Why:** the brief gives A persistence, and accuracy is deterministic counting like `report.py`. It also balances the workload. Subject to Engineer A's confirmation in PR review.

## 9. Known limitations

- AI labels are inferences, not verified facts.
- Small, voluntary samples do not represent all attendees.
- Mocked tests do not prove the real provider is reachable.
- Accuracy on a 12-review gold set is not a general accuracy claim.
