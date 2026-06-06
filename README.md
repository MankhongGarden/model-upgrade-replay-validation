# Model-upgrade replay validation — why a passing benchmark isn't safe to ship

**A benchmark proves your new *model* is better. It does not prove your new *prompt* is better on that model.** Those are two different experiments, and conflating them ships regressions that the benchmark never sees.

This is a 2-stage (+1 cheap pre-stage) validation recipe for swapping the model — or shipping a prompt change — behind a production AI extraction pipeline (think: image/document → structured JSON). It runs entirely on flat-rate coding-agent subagents, so it costs **nothing per run** beyond a subscription you already pay for — no eval-platform bill, no per-token API spend.

> Canonical incident that motivated it: a model swap (call them model **A → B**) benchmarked at **+12pp full-match** on the base prompt → clear "proceed". The same swap, *plus* the new rule-pack we wanted to ship alongside it, **regressed −16pp** on the exact same dataset. Shipping straight off the benchmark would have shipped that regression to users. The replay caught it.

---

## The trap, precisely

A model upgrade in production is almost never *just* a model swap. You usually want to ship a prompt change at the same time — a new appendix, a rule-pack, few-shot examples, a class-routed instruction block. So you run a benchmark, it looks great, you ship the bundle.

The benchmark lied to you — or rather, you asked it the wrong question:

| | What it actually measures | What you assumed it measured |
|---|---|---|
| **Benchmark** | model A vs model B, **on the base prompt** | "B + my new prompt is better" |
| **Reality** | A prompt change that *helped model A* can *hurt model B* | — |

New models have different instruction-following behavior. A rule-pack tuned (even implicitly) against the old model can wash out or invert on the new one. The benchmark can't see this because the benchmark deliberately holds the prompt constant to isolate the model. So you need a **second** experiment that holds the *model* constant (at the new one) and varies the *prompt*.

---

## The recipe

### Stage 0.5 · PoC gate (only when the change adds a non-LLM tool)

If the proposed pathway introduces external tooling (an OCR pre-pass, an STT step, a local model, a rule-based extractor), **do not** run the full benchmark first. Run a 30–60 min proof-of-concept on 5–15 representative samples and measure the tool's *own* fundamental metric (e.g. character/digit recall), not end-to-end accuracy.

- ≥ 80% on the representative subset → proceed to Stage 1
- 50–80% → marginal; tune preprocessing or pivot before investing
- < 50% → abort the pathway, document the negative result, move on

> Real example: "60% of our inputs are typed, plain OCR will nail them at 95%+." PoC on representative samples: **0% recall on handwritten, 21% on printed**. 30 minutes killed a 1–2 day investment. The benchmark's elaborate setup is wasted compute if the tool doesn't work on your corpus at all.

### Stage 1 · Benchmark — isolate the model

Measure model A vs model B on the **same base prompt** (no proposed changes — loading them here confounds the result).

1. Pick a stratified sample (default 50; mix easy/hard, match your class distribution; seed it for reproducibility).
2. Load the **real production** system prompt.
3. Spawn parallel subagents: N on `model: A`, N on `model: B`, base prompt only, 1 attempt per sample, each writing JSON to a file.
4. Aggregate.

**Proceed to Stage 2 only if** Δ full-match ≥ +10pp **and** Δ key-recall ≥ +0.08 **and** Δ aggregate-within-5% ≥ −2pp. Otherwise `MARGINAL` / `ABORT`. A passing Stage 1 is **not** a ship signal — it's a "the model is worth the replay" signal.

### Stage 2 · Replay — isolate the prompt change

Re-use the exact Stage 1 dataset and inputs. Now run `model: B` **+ the proposed prompt change**.

The one thing people get wrong: **compare against the Stage 1 candidate (B on base prompt), not against Stage 1 production (A on base prompt).** Stage 2's only job is to measure whether the prompt change *helps or destroys model B's baseline gain*. Comparing against A re-mixes in the model effect you already measured.

Decision rules:

| Verdict | Condition |
|---|---|
| `CUT_OVER` | Δfm ≥ +3pp vs B-base · Δkr ≥ +0.05 · no subcategory regresses > −5pp → ship the full bundle |
| `PER_CLASS_ROUTE` | some subcategories win, some lose → ship the change only for the winners, keep current behavior for the losers |
| `ITERATE_PROMPTS` | signal in places but not coverage → revise the change, re-replay |
| `KEEP_CURRENT` | the change makes it worse → ship neither; keep A + current prompt |

Average hides this. **Always read the per-subcategory breakdown before a `CUT_OVER`** — an overall +4pp can be +15pp on easy inputs and −9pp on the hard ones you actually care about.

---

## Why it's free: run it on subagents, not an eval API

Do **not** spend paid API quota validating. Dispatch the arms as parallel background subagents under your flat-rate coding-agent plan:

- In one message, spawn 2N agents (Wave A `model: A`, Wave B `model: B`), `run_in_background`.
- Each agent reads the base prompt, reads its assigned input batch, writes one JSON output per sample to disk, and returns a **≤100-word summary** — never paste the JSON into the return (context overflow).
- Don't poll agent status; wait for the completion notification.
- Wall time ~15–25 min per stage.

Stage 1 ≈ 2N agent calls; Stage 2 ≈ N (+ a few for any classifier routing). Cost: $0 incremental.

---

## Four scoring patterns that bite

1. **Recompute aggregates from atomic fields — never trust the model's self-reported total.** If one arm scores against the model's claimed `grand_total` and the other recomputes it from line items, you'll measure a phantom delta that vanishes when both recompute. (A real run showed a fake −21.5pp that was pure scoring asymmetry.)

2. **Don't judge on binary full-match alone when a condensed prompt floors it at 0.** A stripped prompt can score 0/N full-match while line-level recall genuinely differs. Decide on graded recall, not the all-or-nothing metric.

3. **For *small* effects (a single surgical rule, not a model swap), measure your noise floor first.** One read per arm gets buried under read-noise (≈ ±7pp graded, plus the occasional catastrophic read that mangles a whole input). Use ≥3 reads/arm, drop reads with key-recall < 50% (those are catastrophic reads, not your change's effect), compare arm means, and treat within-arm variance as the floor: **any effect smaller than that is unmeasurable — ship it on mechanism, not on an A/B you can't resolve.**

4. **Watch the image/attachment budget.** Vision-input sessions exhaust after ~5–10 large reads per agent. "5 inputs × 3 consensus reads = 15 reads/agent" silently degrades to single-pass on the ones that exhaust. For consensus, use **1 input × 3 reads per agent**, or 3 agents × 1 read.

---

## Validation-set hygiene

If your proposed prompt change was built using the gold set, Stage 2 is measuring overfit, not improvement (few-shot examples lifted from gold are *certainly* overfit; rule-packs tuned on gold corrections are *probably*). Mitigate with a 70/30 build/validate split, or a temporal split (build on inputs before date X, validate on/after), or a second replay on truly held-out items the user labeled *after* the change.

---

## Anti-patterns

- ❌ **Ship straight off the benchmark verdict.** This is the whole point — a passing Stage 1 still needs Stage 2.
- ❌ **Run validation on a paid eval API.** Subagents under a flat plan do it for free.
- ❌ **Compare Stage 2 against production-base instead of candidate-base** — re-mixes the model effect into the prompt measurement.
- ❌ **`CUT_OVER` without reading per-subcategory regressions** — the average lies.
- ❌ **Pre-tune the prompt on items that are in your validation gold** — invalidates the verdict.
- ❌ **Add a prompt rule to fix an edge case that a deterministic post-processor could fix.** Prompt additions wash out (wins and regressions cancel) *and* churn unrelated inputs through distraction — a real layout-rule cratered an unrelated hard subcategory by −48pp. Fix structured output downstream instead, and backtest the rule's fire-precision on gold before implementing it (a 40%-precision heuristic should die before it ships).

---

## When to use this

- Swapping the production reader model (A → B).
- Shipping a non-trivial prompt change (new rule-pack, class router, appendix) — with or without a model swap.
- You have a gold set of ~30+ human-validated correct outputs, stratified.

**Skip it** for trivial prompt edits (a typo, a single-word rule) or when you have no gold set yet — build the gold first; a validation harness with nothing to validate against is theater.

---

## License

MIT — see [LICENSE](LICENSE). If this saved you a bad cut-over and you want production-grade help building the harness for your own pipeline, the [Sponsors](.github/FUNDING.yml) button is there.
