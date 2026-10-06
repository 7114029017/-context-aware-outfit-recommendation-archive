# Known limitations

Everything this artifact cannot reproduce source-exactly, or whose historical
source was not preserved (2026-09-30 guideline, section 十 item 9). For each
item: what is missing or not exact, what the artifact does instead, and where
the evidence is.

Overall: the 2026 results are fully reproducible from this repository (35
training units, statistics and tables, one command). The provenance of the
historical 2025 thesis pipeline is **partial**.

Differences between the latest results and the thesis (for example Recall@1
no longer significant after BH, and the Weather / Occasion OR signs) are not
limitations of the artifact. They are listed in
`docs/manuscript_reconciliation/reconciliation.md`.

## Model and training

| Item | Missing or not exact | What the artifact does | Evidence |
|---|---|---|---|
| CP decoder | The handoff's `DecoderLayerWithCrossAttn` has no definition (import commented out) | Uses the upstream authors' published decoder, `torch.nn.TransformerDecoderLayer`; historical forward behavior not claimed | `standardized_decoder.md` |
| 2025 runtime | Thesis trained on an RTX PRO 6000 Blackwell; exact 2025 package versions not recorded | 2026 runtime recorded (GB10, Python 3.12.3, PyTorch 2.9.1+cu130); bit-identical results are not expected on other GPU / CUDA / PyTorch builds | `environment/`, `reference_run_provenance.md` |
| Hyperparameter search | No tuning records; unknown whether alternatives were tried before adopting the upstream configuration; second-model rationale not recorded | Every setting sourced or marked missing; no supplementary sensitivity analysis | `hyperparameter_tuning.md`, `results/tuning.csv` |

## Data and subsets

| Item | Missing or not exact | What the artifact does | Evidence |
|---|---|---|---|
| Fair-subset membership | Historical memberwise ID file, its SHA-256 and the generation program were not preserved | Reproducibly reconstructs the subset (21,903 / 9,930 test / 3,432 OR queries, same counts as the thesis) and uses it as the formal subset; member identity with 2025 not claimed | `fair_subset.md` |
| Dataset access | Original 2025 download date not recovered | 2026 bootstrap from the `Stylique/Polyvore` mirror, verified by byte-identical split manifests | `data/README.md` |
| Polyvore images | Not redistributed | Image-dependent steps (case figures, color judgments, LLM regeneration) cannot be rerun from this artifact | `data/README.md` |
| Item features | Exact FashionCLIP / SentenceBERT model revisions not established | Uses the preserved feature files (Git LFS, SHA-256 verified); features are not re-extracted | `configs/cp.yaml`, `feature_manifest.md` |
| Description generation | The 35,140 context-aware descriptions and CLO / MET estimates came from Gemma-3-4B-IT; prompts and listed inference settings are preserved, sampling details beyond them are not certified | Uses the preserved generated descriptions; no regeneration | `prompts/semantic_rewrite.txt`, `prompts/clo_estimation.txt`, `prompts/activity_met_mapping.txt` |

## LLM-as-a-Judge and audits

| Item | Missing or not exact | What the artifact does | Evidence |
|---|---|---|---|
| Judge models | Exact Qwen3-VL-32B-Instruct and Gemma-3-27B-it revisions not recovered | Re-analyzes the preserved formal Judge outputs; no Judge rerun | `judge_provenance.md`, `configs/llm_judge.yaml` |
| Judge prompts | Exact P0 / P1 / P2 prompt text not recovered | C / C* checklists preserved; robustness statistics recomputed from preserved outputs | `judge_provenance.md` |
| Robustness sample | Regenerating the 150-record sample yields only 3 / 150 of the archived IDs | Uses the archived 150-record files | `results/summary/reference_20260921T175217Z/secondary/remaining_reproduction_matrix.md` |
| Table 4-7 | Recomputed low-score overlaps 669 / 1,338 / 2,877 vs thesis 668 / 1,323 / 2,833 (cutoff ties; historical inputs) | Reports the recomputed values and the difference | same matrix; `chapter4_table_overview.csv` |
| Table 4-6 | Historical intermediate `reliability_meta_from_subset.csv` not archived | Scope (9,311) and bigram count reproduce; several lexical counts differ and are reported | same matrix |
| Human audit (Tables 4-9, 4-10) | Label-generation provenance not proven | Re-analyzes the 750 archived human judgments; no new judgments | `chapter4_table_overview.csv` |

## Analyses outside the 35 training units

These use historical checkpoints or preserved outputs, not the 2026 models.

| Analysis | Status | Evidence |
|---|---|---|
| Two-Tower second model (TORS Table 8) | Means and SDs recomputed exactly from preserved per-seed outputs; not retrained; CIs not recomputed | `secondary/two_tower/` |
| Counterfactual retrieval (TORS Table 9, 24 cases) | Historical seed-1 checkpoint; not rerun | `03_實驗與結果_experiments_results/06_反事實情境敏感度/` |
| Category and context-subset analyses (thesis 4.5, Figures 4-8 to 4-13; TORS 5.5) | Historical checkpoints and historical fair-subset rows; not recomputed | `03_實驗與結果_experiments_results/04_情境子集與三因子分析/` |
| Case studies and color analysis (Tables 4-17 to 4-20) | Historical rankings; identical-case fresh rankings and image-based judgments not re-executed | `chapter4_table_overview.csv` |

## Records kept as they were

- The run records of `reference_20260921T175217Z` and `full_20260924T102513Z`
  keep their original wording ("temporary … candidate" decoder, "candidate"
  subset). They are historical records and are not edited.
- `results/README.md` and the reference manifest are part of the hashed
  reference result set (`results/SHA256SUMS.txt`) and keep their original
  wording for the same reason.

## Publication items still pending

Public repository access, a fixed release, a DOI (if required), the Artifact
Availability statement and third-party redistribution terms
(`tors_compliance.md`, A1 / A2 / A10).
