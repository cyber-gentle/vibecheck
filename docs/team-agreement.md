# VibeCheck AI — Team Agreement

## Team Structure
- Engineer A: Data Integrity and AI Integration
- Engineer B: Reports and User Interaction

## Communication
- Daily progress updates.
- Regular discussions about blockers.
- Technical decisions documented in the repository.

## Development Rules
- No direct feature commits to `main`.
- Every task must have a GitHub Issue.
- Every feature must have its own branch.
- Every Pull Request must be reviewed by the other engineer.
- Tests must pass before merging.

## Accountability
- Both engineers must contribute meaningful code and tests.
- Both engineers must understand the complete application.
- Both engineers must participate in integration and final testing.
- Each engineer must be able to explain their contributions.

## Conflict Resolution
- Discuss disagreements using technical reasoning.
- Test competing solutions where necessary.
- Document the final decision.
- Escalate unresolved issues to the project supervisor.

## Folder Structure
vibecheck-ai/
│
├── main.py
│
├── app/
│   ├── __init__.py
│   ├── models.py
│   ├── ingest.py
│   ├── provider.py
│   ├── classify.py
│   ├── storage.py
│   ├── cache.py
│   ├── report.py
│   ├── retrieve.py
│   ├── qa.py
│   ├── evaluation.py
│   └── cli.py
│
├── data/
│   ├── synthetic_reviews.csv
│   └── gold_set.json
│
├── tests/
│   ├── test_ingest.py
│   ├── test_classify.py
│   ├── test_report.py
│   └── test_qa.py
│
├── docs/
│   ├── requirements.md
│   ├── architecture.md
│   └── design-decisions.md
│
├── requirements.txt
├── README.md
├── .env.example
└── .gitignore