# The LLM extracts, code scores, and a harness measures the ranking

*Adapted from ADR 0013 of the private repository (LLM pipeline, salary estimate and deterministic scoring) and its amendments.*

## Context

The product has to rank jobs by how worth applying they are. Letting the model output a score directly would be cheap to build, but the score would be opaque, unstable across model or prompt versions, and impossible to re-weight without paying for new calls. Several factors (commute time, employer rating) also arrive later than the facts, from other stages.

## Decision

- **The LLM extracts facts; a pure function scores.** `score(card, facts, settings)` returns 0–100 plus a breakdown that stores each factor's value, weight and reason. Factors: salary against the user's target, location (remote, or hybrid within a radius), benefits, flexible hours, employer rating and commute. A settings change re-scores instantly with no LLM call.
- **Pending factors are renormalised.** A factor that isn't known yet, such as a commute not yet computed, is shown as *pending* and left out, and the remaining weights are renormalised. The alternative (a neutral value) is implemented too. Choosing between them is left to a pre-registered evaluation (below), not to taste.
- **Salary estimates are bounded by official data.** The LLM picks an occupation group from a whitelist. Code derives the level from the extracted education and seniority, and fetches the official wage quartiles for the region. The LLM only places the job between the 25th and 75th percentile, and code clamps the result. The spike showed that the level dominates the result, so the level isn't left to the model.
- **Versioned recipes.** The facts prompt and the score rubric have separate version constants. Stored scores carry the rubric version, and an architecture test forbids an ambiguous shared constant.
- **An evaluation harness on user labels.** Users label jobs `apply` / `maybe` / `no`. `eval report` scores the labelled cards *in memory* from their current facts, rubric, settings and CV, never from stored scores, so one report can't mix rubric versions. It reports Kendall τ-b, Spearman ρ, NDCG@10, precision@10 and pairwise accuracy, and compares the result with the previous report. `eval rerun` re-extracts exactly the labelled set after a prompt or model change.
- **A pre-registered policy comparison.** `eval compare-pending` compares the two pending-factor policies with a seeded paired bootstrap of ΔNDCG@10. Only the pre-registered run (seed 0, at least 1,000 resamples) may decide, and only after a data gate: at least 30 scored labelled cards, 5 `apply`, 5 `no` and 10 with a pending factor.

## Consequences

- Rankings are explainable on the page, because every factor shows its value, weight and reason. They also stay stable across model upgrades, because facts and scoring are versioned independently.
- Tuning weights is free. Changing the prompt or model needs a rerun of the labelled set, and the harness shows the effect.
- The data gate hasn't been met yet: the only run recorded in the repository (2026-09-29) was on test fixtures, with 3 labels. So the default policy (renormalise) stays and is recorded as such. [EVALS.md](../../EVALS.md) describes this state and what comes next.
