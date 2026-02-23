# Paper 3 — Reproducibility Package

**"Continuous Representations, Discrete Commitment: A Causal Analysis of the Commitment Threshold in Transformer LLMs"**

> **Reconstructed bundle.** The original aggregation notebook ran in a temporary Colab
> session and was not saved. This package reconstructs those procedures from the
> per-model intermediate CSVs and from `data/archived_inputs.csv`, which contains
> frozen values (with provenance notes) for GPT-2 and Gemma where the source files
> were not retained. This is NOT a circular reference — archived values were recorded
> from the original run outputs, not re-read from this package's outputs.
> See `validation_report.md` for a full comparison of original vs reconstructed values.

This folder contains everything needed to reproduce the paper's five main result tables
from scratch, or to verify the archived results without re-running the GPU experiments.

---

## Quick orientation

```
reproduce/
├── README.md                              ← you are here
├── requirements.txt                       ← Python dependencies
├── config.yaml                            ← ALL hyperparameters and model IDs
├── data/
│   ├── prompts.json                       ← ALL prompts (convergence, causal, M6)
│   └── archived_inputs.csv               ← frozen M4/M6 values (GPT-2, Gemma)
└── notebooks/
    ├── 01_descriptive_measurements.ipynb  ← M1–M6 per model  (~2–4 GPU hrs)
    ├── 02_causal_swap.ipynb               ← A/B swap test per model  (~1–2 GPU hrs)
    ├── 03_compute_audit_table.ipynb       ← builds cross-model summary table  (CPU, ~5 min)
    └── 04_aggregate_docs_only.ipynb       ← produces paper-ready CSVs  (CPU, ~10 min)
```

Results land in `results/` inside this repo.

---

## 1. Environment setup

```bash
# Python 3.10 or later recommended
pip install -r requirements.txt
```

**Gemma-2-2B only** — requires a HuggingFace account with access approved at
`https://huggingface.co/google/gemma-2-2b`. Then:

```bash
export HF_TOKEN=hf_your_token_here
```

---

## 2. Hardware requirements

| Model | VRAM | Full runtime | Fast mode |
|---|---|---|---|
| GPT-2 Small | 8 GB | ~2–3 GPU hours | ~30 min |
| Gemma-2-2B | 16 GB | ~4–6 GPU hours | ~1 hour |
| Qwen2.5-1.5B | 12 GB | ~3–4 GPU hours | ~45 min |

Fast mode produces approximate results (fewer prompts/iterations) for quick sanity-checking.
Enable it by setting `fast_mode.enabled: true` in `config.yaml`.

All experiments were run on NVIDIA A100 (80 GB) via Google Colab Pro+.

---

## 3. Full reproduction pipeline

Run notebooks **in order**, once per model (change `MODEL_PRESET` at the top of each notebook):

### Step 1 — Descriptive measurements (M1–M6)
Open `notebooks/01_descriptive_measurements.ipynb`.

Change `MODEL_PRESET` to one of `'gpt2'`, `'gemma2_2b'`, `'qwen2_1_5b'` and run all cells.

Outputs saved as `{MODEL_PRESET}_m*.csv` in the notebook's working directory.
They are written to `results/` by default.

**Run once for each of the three models.**

### Step 2 — Causal A/B residual swap test
Open `notebooks/02_causal_swap.ipynb`.

Change `MODEL_PRESET` and run all cells.

Outputs: `{MODEL_PRESET}_causal_swap_raw.csv`, `_layer_summary.csv`, `_checks.csv`.
They are written to `results/`.

**Run once for each of the three models.**

### Step 3 — Build cross-model audit table
Open `notebooks/03_compute_audit_table.ipynb` and run all cells.

This reads the M1–M5 summary CSVs from Step 1 and writes:
```
results/paper3_cross_model_audit_table.csv
```

If source CSVs are unavailable for a model (e.g. you skipped Step 1 for GPT-2),
the notebook falls back to the existing audit table and marks those rows `[archived]`.

### Step 4 — Aggregate to paper-ready tables
Open `notebooks/04_aggregate_docs_only.ipynb` and run all cells.

Reads the audit table + causal swap raw files + any available per-model M4/M6 files.
Writes five CSVs to `results/`:

| Output file | Paper location | Key content |
|---|---|---|
| `paper3_l0_diagnostics_docs_only.csv` | §4.1 | Hourglass ratios, L0 inflation |
| `paper3_invariance_summary_docs_only.csv` | §4.2 | Boundary depth, M6 correlation |
| `paper3_m6_stats_docs_only.csv` | §3.6 | Entropy–convergence Spearman rho |
| `paper3_causal_effect_sizes_docs_only.csv` | §5 | Late/early band delta, bootstrap CIs |
| `paper3_core_metrics_table.csv` | Table 1 | All-in-one summary |

The final cell runs spot-checks against expected values (0.5% tolerance).

---

## 4. Fast verification (no GPU required)

The `results/` directory is where run outputs are written.
To verify the aggregation pipeline without re-running experiments:

1. Ensure the three `*_causal_swap_raw.csv` files are in `results/`
   (already present in the repository).
2. Run **notebook 04 only**.
3. All spot-checks should report `PASS`.

Expected values:

| Model | `late_delta_mean` | `hourglass_ratio_no_l0` | `m6_rho` |
|---|---|---|---|
| GPT-2 | 2.514 | 3.199 | 0.731 |
| Gemma-2-2B | 11.687 | 4.797 | 0.345 |
| Qwen2.5-1.5B | 10.379 | 25.862 | 0.379 |

---

## 5. Compute budget

| Stage | Per model | 3 models total |
|---|---|---|
| M1–M6 descriptive (full) | 2–4 GPU hours | 6–12 GPU hours |
| Causal swap (full) | 1–2 GPU hours | 3–6 GPU hours |
| Audit table (CPU) | 5 min | — |
| Aggregation (CPU) | 10 min | — |
| **Total** | | **~10–18 GPU hours** |

---

## 6. What is archived

Two types of values were computed in the original Colab run but cannot be recomputed
without re-running notebook 01, because the intermediate CSV files were not retained:

| Model | Archived values | Original notebook |
|---|---|---|
| GPT-2 | `hourglass_ratio_no_l0`, M6 convergence/entropy stats | `gpt2_rerun_confirmation_fast.ipynb` |
| Gemma-2-2B | `hourglass_ratio_no_l0`, M6 convergence/entropy stats | `gemma2_2b_comprehensive_from_gpt2.ipynb` |

These are stored in `data/archived_inputs.csv` with provenance notes.
They are **not** read from the output files of notebook 04 (no circular dependency).

If you re-run notebook 01 and save the M4 ablation summary and M6 per-prompt files,
notebook 04 will automatically use the freshly computed values instead of the archive.

**Qwen2.5-1.5B** has all intermediate files available — its values are always live-computed.

---

## 7. Extending to new models

To add a model (e.g. Mistral-7B):

1. Add an entry to `config.yaml` under `models:`:
   ```yaml
   mistral_7b:
     hf_id: "mistralai/Mistral-7B-v0.1"
     n_layers: 32
     expected_late_band: [19, 24]   # Eq1 prediction: C ≈ 0.57×32+2.3 ≈ 21
     raw_late_floor: 0.55
   ```

2. Run notebooks 01 and 02 with `MODEL_PRESET = 'mistral_7b'`.

3. Run notebook 03 — the new model will be detected from the new CSVs.

4. Run notebook 04 — spot-check thresholds are model-agnostic.

---

## 8. Prompts

All prompts are in `data/prompts.json`. Three sets:

- **`convergence_prompts`** (20): open-ended completions for M1–M5 descriptive analysis
- **`m6_difficulty`** (15 = 5 easy + 5 medium + 5 hard): certainty-stratified prompts for M6
- **`causal_pairs`** (28): factual A/B pairs for the causal swap; filtered per tokenizer
  to single-token targets at runtime (typically 15–24 pairs survive per model)

---

## Citation

If you use this code or data, please cite:

```
[Citation to be added upon publication]
```

---

## Contact

For questions about reproducing these results, open an issue or contact the authors.
