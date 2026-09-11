# Portion 10B — submission packaging and clean setup

Completed 11 September 2026.

## Delivered

- Added a root `README.md` with the exact Ola track, plain-language system flow, project map, dataset choices, measured results, limitations, Windows setup, API examples, privacy boundaries, and submission review note.
- Added `run_server.py` so the local API starts with `python run_server.py` and binds to `127.0.0.1:8000`.
- Added `uvicorn==0.52.4` explicitly to `requirements.txt`.
- Closed the request-logging gap with an HTTP fallback that safely covers malformed JSON, validation errors, framework routes, unmatched URLs, and unexpected failures without duplicating application-route records.
- Added logging tests for invalid JSON, a missing field, whitespace, an unknown ticket, an unmatched URL, and a phone-shaped session ID. Every case creates exactly one record and raw test identifiers do not survive in the log.
- Added the Portion 10A audit regression probes to the full-project runner.

## Clean setup evidence

A new copy was created without `.venv`, `models`, or `storage`. In that separate copy:

1. A new Python 3.12 virtual environment was created.
2. CPU PyTorch 2.14.0 and every pinned requirement were installed from package sources.
3. `pip check` reported no broken requirements.
4. The pinned `all-MiniLM-L6-v2` revision was downloaded using `prepare_model.py`.
5. Both Chroma collections were rebuilt: 31 fixed chunks and 36 sentence chunks, each 384-dimensional.
6. `verify_full_project.py` passed all 14 commands. The main suite ran 44 tests, the dataset/retrieval suite ran 11 tests, and the targeted audit probes also passed.

The host used an environment-specific proxy adjustment only to let the clean model download reach its public source. That adjustment is not part of the project or needed on an ordinary internet connection. Windows instructions remain unexecuted on Prashant's laptop and are labelled accordingly in the Task 3 document.

## Result

The local repository is ready for final publication preparation. The final public GitHub URL and LMS submission are separate account actions and are not claimed here.
