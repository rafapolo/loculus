# 01 · Ingest: fix the quadratic lookup in `prepare_files.py`

> **Solves:** Cuts ingest file preparation for 200k-sequence organisms from 15+ minutes to seconds by replacing O(n) list lookups with O(1) set lookups.

| Field | Value |
|---|---|
| Issue | [#7443](https://github.com/loculus-project/loculus/issues/7443) |
| Component | ingest |
| PR title | `perf(ingest): use sets for submit/revise/revoke lookups in prepare_files` |
| Effort | XS (≈ 1–2 h including benchmark) |
| Risk | Very low: same output, different data structure |
| Status | todo · 🟢 free |

## Existing work (checked 2026-11-23)
Issue #7443 is open and unassigned, with **no linked PR** and no PR touching `prepare_files` (checked 2026-11-23). Comment on the issue before starting.

## Problem
Preparing files for a large organism takes **15+ minutes for 200k SARS-CoV-2 submissions**.

## Root cause (verified)
- `ingest/scripts/prepare_files.py:106-108` loads `to_submit`, `to_revise` and `to_revoke` with `json.load`, so they are **lists**.
- The main loop (`for field in orjsonl.stream(metadata_path)`, line 129) runs `if fasta_id in to_submit:` (line 135), and then the same check against `to_revise` and `to_revoke`.
- Each membership test is O(n), so the whole step is O(n²): about 4·10¹⁰ comparisons for 200k records.

## Plan
1. Convert the three collections to `set(...)` right after loading them. If the JSON is a dict, keep the dict, since its key lookup is already O(1); check the producer, `compare_hashes.py`.
2. Look in the same file for other list membership checks or `list.index` calls, and in `call_loculus.py` and `compare_hashes.py` for the same pattern.
3. Keep the output order identical, because the TSV is written in the streaming order of `metadata_path`, not in the order of `to_submit`.

## Acceptance criteria
- Byte-identical output files (`metadata_submit`, `metadata_revise`, `sequences_*`) on the existing test fixtures.
- A benchmark in the PR description: list vs. set with 200k synthetic IDs (`python -m timeit` or a small script), and the end-to-end time of the rule on a large fixture.
- `ingest/tests` passes; `ruff` is clean.

## Test plan
- Run `pytest` in the `ingest` conda environment.
- Generate a synthetic `metadata.ndjson` with 200k records plus matching `to_submit.json`, time it before and after, and `diff` the outputs.

## Files
- `ingest/scripts/prepare_files.py`
- `ingest/tests/` (add a regression test with a few thousand IDs if there is none)
