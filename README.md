# Ola — Business Operations / Customer Support

This capstone is a working demonstration of a customer-support assistant for the **Ola — Business Operations / Customer Support** track. It answers questions from a fictional support handbook, checks fabricated support tickets, remembers a ticket within one chat session, masks supported personal-data formats, and reviews every grounded answer before returning it.

The system does not connect to Ola, identify real customers, approve refunds, change tickets, or take financial actions. A human support employee remains responsible for those decisions.

## What happens when someone asks a question

1. The input check masks supported phone, card-last-4, PAN, Aadhaar, and labelled bank-account formats. It blocks obvious prompt-injection instructions.
2. Session memory resolves a follow-up such as “What is its status?” when the earlier turn named a ticket.
3. CrewAI runs three workers: Retrieval finds handbook text, Lookup checks fabricated ticket records, and Composer creates a typed answer.
4. A Pydantic check rejects missing or unsupported response fields.
5. Two AutoGen reviewers compare the draft with the original evidence and either approve it or correct it.
6. FastAPI returns the answer and writes one masked JSON-Lines log record with a trace ID and timing.

The language models used by CrewAI and AutoGen are deterministic `MOCK_LLM` implementations. They make the project free and repeatable while still exercising the real framework workflows. Policy search uses a real local SentenceTransformers model and two real ChromaDB collections.

## Project contents

| Item | Purpose |
| --- | --- |
| `dataset.py` | Generates and validates 50 fabricated support tickets. |
| `knowledge_base/` | Contains 12 original fictional support-policy documents. |
| `rag_index.py` | Builds and searches fixed-size and sentence-based Chroma indexes. |
| `crew_workflow.py` | Runs the three CrewAI workers and their tools. |
| `session_service.py`, `guardrails.py` | Provide session memory, masking, injection checks, and grounded-output checks. |
| `autogen_review.py` | Runs the bounded two-agent answer review. |
| `api_app.py`, `run_server.py` | Provide HTTP and WebSocket access. |
| `evaluation.py`, `verify_*.py`, `test_*.py`, `tests/` | Run evaluation, integration checks, and focused tests. |
| `transcripts/` | Stores real machine-readable execution evidence. |
| `TASK*.md`, `AUDIT10A.md` | Explain design choices, results, and audit limits. |

## Dataset design

The generator uses random seed `3`, so the same 50 records are recreated every time. Category weights are Billing 30, Technical Issue 25, Account Access 15, Product Defect 10, and General Inquiry 20. Status weights are Open 20, In Progress 25, Escalated 15, Resolved 25, and Closed 15. These are generation weights, not guaranteed final percentages.

The generated dataset contains at least three records in every required category and at least one in every status. Ten of 50 records have an escalation flag, giving the required 20% escalation rate. Resolution time is an integer from 1 to 72 hours and ticket age is an integer from 0 to 30 days. Active-ticket resolution hours are estimates. The Ola ticket schema does not contain an amount field, so an unrelated amount range was not added.

All tickets and policy documents are fictional and contain no real customer data or real Ola policy claims.

## Windows setup

Use 64-bit Python 3.12. The one-time setup needs internet access to install packages and download the pinned embedding model. After that, the project verification removes API keys and runs the model and indexes locally.

Open PowerShell in the project folder and run:

```powershell
py -3.12 -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install torch==2.14.0 --index-url https://download.pytorch.org/whl/cpu
python -m pip install -r requirements.txt
python prepare_model.py
python rag_index.py build
python verify_full_project.py
```

If PowerShell blocks activation, run `Set-ExecutionPolicy -Scope Process Bypass` in that terminal and activate again. If the `py` launcher is unavailable, replace `py -3.12` with the command for your Python 3.12 installation.

Installing CPU PyTorch first avoids downloading unnecessary GPU packages. A successful final command prints `"status": "PASS"`. It runs the dataset generator, every saved portion verifier, the audit regression probes, both test suites, and the dependency check. It also refreshes `transcripts/full_integration_evidence.json`.

## Start and try the API

With the virtual environment active, run:

```powershell
python run_server.py
```

Open `http://127.0.0.1:8000/docs` in a browser. Expand `POST /ask`, choose **Try it out**, and use:

```json
{
  "session_id": "demo-1",
  "message": "How often should outage progress updates be sent?"
}
```

For a ticket example, change the message to `Please check ticket SUP-0014.` The response includes the final answer, evidence-based ticket fields, review verdict, trace ID, and processing time. Use `POST /sessions/demo-1/reset` to clear that session. The WebSocket chat endpoint is `/ws/chat/{session_id}`.

The server binds only to `127.0.0.1` for a local demonstration. Stop it with **Ctrl+C**.

## Measured results

- Both Chroma indexes use the pinned local `all-MiniLM-L6-v2` model with 384-dimensional normalized embeddings and cosine distance.
- Fixed-size splitting created 31 chunks; sentence splitting created 36. Sentence splitting was selected after a 12-query comparison: macro precision `0.6250`, macro recall `1.0000`, and the expected first source on 12 of 12 selected questions.
- The demonstrated fallback threshold is `0.40`, selected from measured in-scope and out-of-scope examples.
- Ticket attention uses `0.65 × escalation flag + 0.35 × normalized age`, with a demonstrated threshold of `0.50`.
- The 15-case acceptance evaluation covers all 12 handbook topics plus a fabricated ticket, an out-of-scope question, and an injection attempt. Its deterministic judge gives `5.00/5` averages for Accuracy, Grounding, Completeness, and Safety on that selected set.

Detailed evidence is in `transcripts/portion9_evaluation.json`, `transcripts/full_integration_evidence.json`, and the task documents.

## Limits of the demonstration

- The evaluation is a selected acceptance set, not held-out testing or proof of general accuracy. For example, “Who handles a suspected account takeover?” scored `0.388068` and fell back, while the more explicit evaluated wording passed.
- The judge is deterministic and rule-based. Its recorded prompt and checks suit this extractive mock but do not validate arbitrary paraphrases as an independent real LLM would.
- Runtime token and cost figures are a preflight reference simulation because all LLM calls are free mocks. The 768-token reserve is not measured total framework consumption; a paid deployment would need per-call accounting and enforced billing limits.
- Class-level traces and in-memory caches support a single demonstration process. One API app serializes chat processing; isolation across multiple processes or app instances has not been established.
- The assistant has no authentication, customer-ownership checks, live Ola access, financial-action permissions, arbitrary PII detection, or automated retention deletion.
- Unknown tickets return a controlled generic error. Policy answers depend on wording and the measured similarity threshold.

## Privacy and runtime choices

Request logs contain masked input, route information, outcome, a random trace ID, and timing. They do not deliberately store raw request bodies. Only the documented fixed formats are masked; names, addresses, and arbitrary free text are outside that guarantee. Never use real customer information in this project.

`verify_full_project.py` removes API-key and proxy variables for the graded run, disables CrewAI and OpenTelemetry telemetry, and checks installed dependencies. The prepared embedding model and indexes must already exist because runtime retrieval is offline.

## Review before submission

The student should run the setup and examples, read `START-HERE.md`, and be able to explain the six-step flow above in their own words. The repository should be submitted under the exact track name shown at the top. Course clarification supplied with the project is the working basis for AI-assisted development; the student remains responsible for reviewing, understanding, and accurately describing the submitted work.
