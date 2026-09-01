# MCSU-Bench

MCSU-Bench is a benchmark for evaluating whether models can recover a complete,
role-labeled vulnerability chain. Each scored example is a **Minimum Complete
Semantic Unit (MCSU)**: a causally sufficient, locally minimal unit containing
the Source, required Propagation steps, Sink, and the context needed to
interpret them.

The repository contains the construction pipeline, canonical reference files,
frozen experiment artifacts, paper-generation scripts.

## What This Repository Supports

- Construction of auditable MCSUs from verified vulnerable revisions.
- Exact Source/Propagation/Sink/Guard recovery evaluation for multiple code
  representations and model endpoints.
- Re-scoring existing predictions after an in-place canonical Gold correction.
- Regeneration of paper tables, figures, and numeric consistency checks from
  versioned artifacts.

The fixing patch is used only during offline construction. It is never included
in an evaluation prompt.

## Repository Layout

```text
src/                 Phase 1--4 construction and evaluation implementation
scripts/             construction, audit, analysis, and downstream rerun CLIs
codeql_queries/      CodeQL queries used by the static-analysis pipeline
dataset/
  labels/            the sole canonical Gold file for each task
  artifacts/         construction evidence, predictions, scores, and analyses
  metadata/          sample metadata and frozen splits
  analysis/          isolated, non-publishable exploratory analyses
tests/               regression tests
```

## Requirements

- Python 3.10 or later.
- Python packages in `requirements.txt`.
- CodeQL 2.23.6 for Phase 1/2 construction and canonical reconstruction.
- `latexmk` with a standard LaTeX installation to compile the manuscript.

API credentials are required only when launching a new model experiment. They
are not required to run the tests, audit the canonical Gold contract, verify
the frozen results, or build the paper. The client reads credentials from the
environment, including `DEEPSEEK_API_KEY`, `GLM_API_KEY`, or `OPENAI_API_KEY`;
do not place credentials in repository files.

## Quick Verification

The following commands validate the checked-in evidence without making model
API calls or changing canonical Gold.

```bash
python3 -m venv .venv
. .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt

PYTHONPATH=. pytest -q
python scripts/audit_single_gold_contract.py
python paper/generate_result_tables.py
python paper/verify_numbers.py
```

The test suite exercises the construction and evaluation code. The Gold audit
checks that each task has exactly one canonical label file and rejects parallel
label namespaces. `verify_numbers.py` checks every reported paper number
against its source artifact and the generated-table manifest.

## Canonical Gold and Downstream Re-scoring

The canonical files are:

- Task 1: `dataset/labels/trace_gold.jsonl`
- Task 2: `dataset/labels/rootcause_fixlogic_gold.jsonl`

After an approved in-place correction to either canonical file, re-score the
existing predictions with:

```bash
python scripts/rerun_downstream_from_gold.py
```

This command is intentionally downstream-only: it audits the single-Gold
contract, re-scores existing predictions, refreshes derived analyses and paper
artifacts, and records the resulting Gold hashes. It does not generate Gold,
overwrite the canonical file, or make LLM calls.

## Rebuilding Construction Artifacts

Full Phase 1/2 reconstruction requires the repository checkouts and the pinned
external phase snapshot. Before rebuilding Phase 4 or freezing a new snapshot,
validate that those external assets resolve consistently:

```bash
python scripts/validate_phase_snapshot.py --require-repos
```

The validator checks Phase 1/2 artifacts and checkout consistency without
reading or modifying canonical Gold. The construction pipeline must keep the
patch diff out of evaluation prompts, and only deterministically rechecked
evidence may enter the canonical MCSU chain.

## Reproducibility Boundaries

- `dataset/artifacts/` is the source of record for paper numbers.
- Main scores use only the frozen main cohort; diagnostic, incomplete,
  unresolved, and missing-prediction cases are excluded from those scores.
- `dataset/analysis/` contains isolated exploratory work marked
  `do_not_publish=true`; it must not be used to select the main prompt or make
  paper claims.
- Model claims must use the provider's actual returned model identifier and
  retain the request configuration, seed-forwarding status, API failures, and
  Gold hash in their run metadata.
