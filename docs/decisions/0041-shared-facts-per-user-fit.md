# CV-free job facts, shared, and a per-user fit made lazily

*Adapted from ADR 0041 of the private repository.*

## Context

Originally one extraction call per card carried the user's CV and preferences in its system prompt and returned about 23 fields. Only two of them, a "why it fits" summary and red flags, judged the job against the candidate. Because the CV was part of the cache key, two users could never share an extraction, and every CV edit re-extracted every card. Sharing results between installs (and later between users of the hosted mode) needed a result that depended on the posting alone.

## Options

| Option | Trade-off |
|---|---|
| One call, share only the posting fields | The CV still colours the "shared" fields (seniority, stack), and the key still holds the CV, so nothing is ever reused |
| **Two calls: posting facts (shared) and fit (personal, lazy)** | Facts become shareable, and a CV edit re-runs only fits. Most cards never need a fit |
| A deterministic, templated fit | No call, but the text *is* the feature |

## Decision

- **Facts** carry no candidate profile and no preferences. Their staleness key is the recipe: model, prompt version, instructions, rubric, schema digest and validation version. The CV is no longer part of it.
- **Fit** is a separate small-model call: the CV and preferences go in the system prompt, and the same posting block goes in the user message (so the input hash is shared). It returns only `summary_fit` and `red_flags`.
- **A fit is made only when it pays off.** Fits are prefetched at the end of the scoring chain for cards scoring at least 70 (best first, at most 25 per run, within the automatic budget and outside peak-price windows). Otherwise a fit is made when the user opens a card that has none.

## Consequences

- Facts can be cached across installs through a content-addressed, first-write-wins *commons* (a Cloudflare Worker with D1, or a self-hosted server), so each posting is extracted once for everyone.
- A user pays for facts only on commons misses, and for fits on roughly the 25 best new cards a day plus the ones they open.
- A CV edit makes only fits stale, and those are re-made lazily.
- The split was a prerequisite for the hosted edition's cost model: the platform pays for shared facts, and each user's budget pays for their own fits.
