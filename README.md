# mini-llm-jepa

Laptop-scale test of the LLM-JEPA idea on a small NL-to-SQL domain.

## The Question

The LLM-JEPA paper adds an auxiliary representation-prediction loss on top of standard supervised fine-tuning and reports a clean gain over an SFT baseline. Those experiments are credible, but they run at scales most laptop and edge deployments cannot touch.

This repo asks the narrower, deployment-shaped version:

> Does the same mechanism survive at 0.5B parameters, on consumer hardware, in a way that would matter for small-model deployment?

The target model is `Qwen/Qwen2.5-0.5B-Instruct`. The task is 14,000 templated NL-to-SQL pairs over a 25-table commercial insurance schema. The point is not to claim broad semantic parsing generalization; the point is to test whether the auxiliary representation objective produces pair-specific structure that SFT alone does not.

## Current Read

This is no longer just setup plumbing. Phases 01-03 have produced a fairly clear diagnosis:

- SFT teaches the model the schema and query shape, but strict exact match remains 0/100.
- SFT moves prompt and target representations together globally, but does not create a useful aligned-vs-shuffled pair gap by itself.
- The first JEPA pass does not improve downstream generation: base, SFT, and JEPA all score 0/100 exact match on the held-out generation eval.
- The corrected Phase 03 audit shows the target manifold is not razor-thin after all. A family-diverse held-out audit has real between-row variance and a small pair-specific raw signal.
- The JEPA predictor still collapses directionally: it learns a common direction better than it learns the row-specific mapping.

Phase 04 answered the identification question: on base Qwen, the pair-specific signal exists but is dominated by a common direction plus family-mean structure. Raw representation gates rejected every candidate, FWL-residualized gates passed everywhere, and the winning candidate (`layer=-1`, `pooling=content_mean`) shows OOD-family FWL R2 = 0.82. That points to Residual JEPA rather than raw JEPA as the next training pass to run.

## Notebook Map

### `01-sft-baseline.ipynb` - SFT Baseline

Plain supervised fine-tuning, no JEPA. It sets up the chat-template/loss-mask contract, trains on the 14k NL-to-SQL corpus, and evaluates held-out generation.

Latest committed read:

- Strict exact match: 0/100.
- Correct schema tables: 96/100.
- Mean token overlap: 74%.
- Near-verbatim predictions above 90% token overlap: 36/100.
- Truncation/repetition failures: 0/100.

Interpretation: real structural learning happened, but strict exact match is too brittle to capture it. The model learned the schema and SQL shape, while still missing enough exact column/order/detail choices to score zero under byte-level matching.

### `02-representations.ipynb` - Representation Audit

Hidden-state extraction and geometry checks for base vs SFT. This notebook measures whether each prompt representation sits closer to its true target than to shuffled targets.

Latest read:

- SFT raises cosine similarity broadly.
- Aligned and shuffled similarities rise together.
- Vanilla SFT does not create the pair-specific representation gap JEPA would need to exploit.

Interpretation: if a later JEPA run creates an aligned-vs-shuffled gap, that gap is not already explained by SFT alone.

### `03-jepa-loss.ipynb` - Minimal JEPA Attempt

First JEPA training pass on top of the SFT checkpoint, with generation eval and representation diagnostics.

Latest aligned-mode rerun:

- Generation eval remains 0/100 exact match, same as base and SFT.
- Corrected audit sample is family-diverse instead of the first 200 pairs from one SQL family.
- Target per-dimension std is about 1.01, not 0.27.
- Target norm std is about 0.47, not 0.18.
- Raw aligned-vs-shuffled gap is about `7e-4`.
- Predicted aligned-vs-shuffled gap is about `1.8e-4`.
- Predictor mean pairwise cosine is about 0.9924, or roughly 7.1 degrees average angle.

Interpretation: the earlier "architecture gives zero pair signal" read was too pessimistic because the audit sample was template-family biased. There is a small pair-specific signal in the raw geometry. The problem is that the predictor mostly learns the common direction and leaves the row-specific signal underused.

### `04-identify-then-intervene.ipynb` - Identify, Gate, Then Intervene

Preflight lab that decides which JEPA variant the next training notebook should run. Extracts a latent panel from base Qwen across candidate `(layer, pooling)` combinations, tests each for geometry, retrieval, frozen linear predictability, and FWL-residualized predictability, then applies a two-lane selection gate (raw vs Frisch-Waugh-Lovell). Machine-readable branch label lands in `04-outputs/geometry/selection_gate_{hash}.json` as `gate_branch`: `none`, `raw_jepa`, `residual_jepa`, or `raw_and_residual`.

Latest run (base Qwen, 3,000 train + 750 calibration + 750 test):

- 279 template families, effective balanced-family count 11.7 (matches the reviewer's Herfindahl estimate).
- Raw gate: 0/12 candidates pass. Ridge OOS R2 beats the null trivially, but every candidate fails on either the constant-predictor cosine advantage or the effective-rank ratio against nb03's terminal-pooling baseline. The raw target geometry is dominated by a common direction and by family-mean structure.
- FWL-residualized gate: 12/12 pass. After partialling out family means from both views, every candidate has recoverable pair-specific signal.
- Selected candidate: `layer=-1, pooling=content_mean`. Ridge R2 all/iid/ood = 0.87 / 1.00 / 0.63. FWL R2 all/iid/ood = 0.92 / 0.98 / 0.82. Retrieval MRR on ridge predictions = 0.93. Target intraclass correlation = 0.964 (~96% of target variance is between-family, ~4% is pair-specific residual).

Interpretation: the pair-specific signal exists and is generalizable to unseen SQL templates (OOD FWL R2 = 0.82), but it is swamped in the raw geometry by the common + family components. That is the exact regime Residual JEPA is designed for. `gate_branch = residual_jepa`. The next training notebook should build the Residual JEPA arm around this `(layer, pooling)` and sweep the number of removed common components on calibration only.

## Run Order

Run the notebooks in order:

```bash
jupyter lab
```

1. `01-sft-baseline.ipynb`
2. `02-representations.ipynb`
3. `03-jepa-loss.ipynb`
4. `04-identify-then-intervene.ipynb`

Install requirements as needed:

```bash
pip install torch transformers huggingface_hub matplotlib pandas numpy scikit-learn jupyter ipykernel
```

The base model is downloaded into `./model/` on first use. Per-notebook artifacts land in `./01-outputs/`, `./02-outputs/`, `./03-outputs/`, and `./04-outputs/`.

## How To Read Success

The bar is not "does Qwen 0.5B become a strong general NL-to-SQL model?" This dataset is templated and schema-specific.

The useful research question is:

> Does a JEPA-style auxiliary loss recover pair-specific prompt-to-target structure that SFT does not already learn?

Phase 04 is the selection gate for that question:

- If raw and residualized R2 both fail, the next step is probably data or objective redesign, not another JEPA training run.
- If only FWL-residualized R2 passes, the next training notebook should test Residual JEPA.
- If both raw and residualized R2 pass, the next training notebook should run both arms as an ablation.
- If raw passes cleanly, a simpler raw JEPA retry is defensible.

## License

MIT. See `LICENSE`.

Adrian Sclafani
