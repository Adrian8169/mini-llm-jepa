# mini-llm-jepa

Laptop-scale test of the LLM-JEPA idea.

## The question

The LLM-JEPA paper added an auxiliary representation-prediction loss on top of standard supervised fine-tuning and reported a clean gain over the SFT baseline. Sharp controls, credible result. Their experiments ran at parameter counts most engineers can't touch (billions of parameters, multi-GPU clusters).

The question this repo asks:

> Does the same mechanism survive at 0.5B parameters, on consumer hardware, in a way that would matter for small-model deployment?

If it holds, a specialized 0.5B model becomes practical for places where a 7B one can't fit: edge inference, cheap per-request serving, tight VRAM budgets.

## What's in here

Two notebooks. Each one is standalone and parameter-driven. A dashboard cell near the top exposes every knob you might want to tune (learning rate, scheduler shape, freezing depth, pooling method, layer choice), and every cell below reacts to whatever you set.

**`01-sft-baseline.ipynb`.** Plain supervised fine-tuning, no JEPA yet. Downloads the base model, sets up the loss-masking contract (only the target tokens contribute to the loss, not the prompt tokens the model isn't supposed to reproduce), then trains on 14,000 templated NL-to-SQL pairs over a 25-table commercial insurance schema. Each epoch pulls a fresh, non-overlapping slice from the training pool. That way, when the loss plateaus on brand-new data, it means the model has reached its ceiling for this training budget, not that it's memorizing what it has already seen. The notebook reports pre/post exact-match on a 100-pair held-out eval, plus a sample-level diagnostic that catches the structural learning the strict metric misses.

**`02-representations.ipynb`.** A hidden-state extraction audit. Pools the prompt and target representations, measures the aligned vs shuffled cosine gap (do prompts sit closer to their own targets than to random targets from the same batch?), and logs collapse diagnostics (whether all the vectors are getting squashed into a narrow region of space, which would be a bad sign). No training loss here yet. This notebook only observes. Its job is to establish the baseline geometry that any future JEPA experiment has to beat.

Target model: `Qwen/Qwen2.5-0.5B-Instruct`, auto-downloaded from HuggingFace on first run into `./model/`. Per-notebook artifacts land in `./01-outputs/` and `./02-outputs/`. Reference outputs from my own runs are committed so you have something to compare against.

## Run it

```bash
pip install torch transformers huggingface_hub matplotlib jupyter ipykernel
jupyter lab
```

Open `01-sft-baseline.ipynb` and run all cells. About 70 minutes on a laptop-class GPU (RTX 4060 or similar), several hours on CPU. Then `02-representations.ipynb`, which is fast because it only extracts and analyzes hidden states rather than training anything.

## What "success" means here

I'm not claiming Qwen 0.5B becomes a strong NL-to-SQL model. The dataset is 14,000 templated queries over one insurance schema, and the held-out eval slice is drawn from the same template families as the training data. So the eval is really asking "did the model learn to fill in these query templates correctly?" more than "can the model generalize to totally new query patterns?"

What Phase 01 does teach the model is the specific schema (actual table names like `policies`, `carriers`, and `accounts`), the aliasing conventions the training data uses (`p.` for policies, `a.` for accounts, `ca.` for carriers), and the PostgreSQL idioms in the target queries.

Strict exact-match on the held-out eval comes back 0/100. That number is misleading, and the sample-level diagnostic in Phase 01 explains why:

- **96/100** predictions reference the correct schema tables (the base model, before training, referenced zero of them; it was inventing fake table names)
- **74%** mean token overlap between prediction and target
- **36/100** predictions have over 90% token overlap with the target (near-verbatim)
- **0/100** truncated outputs or repetition loops (the model always finishes its query cleanly)

Real structural learning happened. The model learned the schema and the shape of the queries. Strict exact-match couldn't see it because that metric wants byte-for-byte string match on the whole query, and the model doesn't always pick the exact same column subset or ORDER BY tail as the target.

Phase 02 adds a second finding. Vanilla SFT doesn't push prompt and target representations into alignment on its own. After training, "aligned" cosine similarity (each prompt vs its actual target) and "shuffled" cosine similarity (each prompt vs a random target from the batch) both rise by roughly the same amount. Which means SFT is pulling every pair closer to every other pair uniformly, without preferring the correctly matched pair over a random one. If a future JEPA experiment produces an alignment-specific gap (aligned rising while shuffled stays put), that gap has to be attributable to the auxiliary loss. SFT alone can't create it.

The question this repo isn't yet in a position to answer is whether JEPA's auxiliary loss can do something a vanilla SFT baseline cannot at this scale. Phase 01 builds the honest baseline. Phase 02 builds the honest measurement plumbing. Any future JEPA experiment now has a fair, credible comparison to work against.

For now, this is the setup, not the verdict.

## License

MIT. See `LICENSE`.

Adrian Sclafani
