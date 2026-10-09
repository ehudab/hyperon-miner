# A Frequency-Based News Recommender in MeTTa on MIND

*Draft. Items marked **[TODO]** are values I did not have and need to be filled in from your runs.*

## 1. Summary

We implement a frequency-based news recommender in MeTTa/PeTTa on the MIND dataset. Article-to-article association scores are computed from user click behavior (mined with the Hyperon pattern miner), lifted to category-level scores, and combined to rank the candidate articles of an impression. On a small evaluation of 101 dev impressions the recommender obtained **AUC = 0.501**, which is indistinguishable from random ranking. We also found that memory use becomes a bottleneck on larger datasets. This report documents the method, the result, and the diagnostics needed before drawing any conclusion about the approach.

## 2. Method

**Data.** Click behavior from MIND **[TODO: dataset version (small/large) and number of training behaviors, e.g. the 1k subset]**. Counts and patterns are built from the *training* behaviors only. Evaluation uses *dev* impressions, each with a user history and a list of candidates labeled clicked / not clicked.

**Mining.** The Hyperon pattern miner finds pairs of articles clicked by the same user above a minimum support **[TODO: min-support and depth]**.

**Directed confidence.** For each surviving pair (A, B), with `c(A)` = number of users who clicked A and `s` = number who clicked both:

- `conf(A → B) = s / (c(A) + λ)`, with smoothing λ = 10
- `conf(B → A) = s / (c(B) + λ)`
- Jaccard `s / (c(A) + c(B) − s)` is stored with the pair but not used in ranking.

Self-pairs (A → A) are dropped.

**Category score.** Each article-pair confidence is mapped to the pair of categories of the two articles. Scores that share the same (category → category) pair are averaged. Because this is derived from the surviving article pairs, a category pair exists only if at least one article pair between those categories survived mining.

**Scoring a candidate.** For one history article `h` and candidate `c`:

```
pair(h, c) = 0.7 · conf(h → c) + 0.3 · catscore(cat(h) → cat(c))
```

A missing article pair or category pair contributes 0 (an unknown category also scores 0). The candidate score is the mean of `pair(h, c)` over the user's history, with repeated history articles counted each time. An empty history gives every candidate a score of 0.

**Metric.** AUC per impression: the share of (clicked, not-clicked) candidate pairs where the clicked one scores higher, with ties counting 0.5. Impressions with no clicked or no not-clicked candidates are dropped (AUC is undefined), and the mean is taken over the remaining impressions.

## 3. Results

| Setting | Impressions | AUC |
|---|---|---|
| Full pipeline (0.7 item + 0.3 category, λ = 10) | 101 | **0.5009** |

Run output: `(EVAL-RESULT (AUC 0.5009226940710074) (valid-impressions 101) (total-impressions 101))`

**Interpretation.** An AUC of 0.50 means the scores carry no usable ranking signal on this evaluation. With 101 impressions the estimate is also noisy: if per-impression AUC has a standard deviation of roughly 0.3 (an assumption, to be measured), the standard error is about 0.03, so the result is within noise of random. We therefore do not claim that the method works or that it fails in general, only that this run shows no evidence of signal.

**Likely explanations to test (not yet verified):**

1. **Low coverage.** Most candidates have no article pair with any history article. Their scores tie at 0, ties count 0.5, and AUC drifts toward 0.5. With a small training slice this is the most likely cause.
2. **The category score is weak.** It is averaged over surviving article pairs and is capped at 0.3 of the blend, so it only separates candidates that lack item evidence.
3. **A pipeline problem.** A bug would also produce 0.5. This needs to be ruled out before anything else.

**Diagnostics to run before the final report:**

- Report **coverage**: the fraction of candidates with a non-zero item score, and with any non-zero score.
- Add **baselines** on the same impressions: random (expected ≈ 0.5) and global popularity.
- Run an **ablation**: item only, category only, combined.
- Sanity check on a few *training* impressions, where the counts include the clicks being scored. AUC should be clearly above 0.5; if it is not, there is a pipeline bug.
- Add MRR and nDCG@5/10 and a bootstrap confidence interval over impressions.
- Repeat on more impressions and a larger training slice.

## 4. Scalability: memory

On larger datasets the pipeline hits a memory bottleneck. **[TODO: quantify. Record peak memory and runtime at each dataset size, e.g. with `/usr/bin/time -v`, and put the numbers in a table.]**

Design features that likely contribute (hypotheses until measured):

- All derived atoms (both directions of every pair, plus category atoms and candidate scores) are stored in a single in-memory space.
- The number of article pairs grows roughly quadratically with the number of articles each user clicked.
- Confidence computation counts users with repeated queries against the behavior space for every pair.

Options to reduce memory: raise the minimum support so fewer pairs survive; cap the history length per user (`--max-history` in `make_impressions.py`); keep the category atoms in a separate space from the article pairs; evaluate impressions in batches rather than all at once; drop the Jaccard value from the stored atoms.

## 5. Limitations

- One small evaluation (101 impressions) with no baselines yet.
- No content or semantic signal, only co-click frequency.
- Missing evidence is scored as 0, so "never seen together" and "seen to be unrelated" are not distinguished.
- The category score exists only where article pairs survived mining.
- No recency weighting of the history.

## 6. Next steps

1. Run the diagnostics in Section 3 and fill in the TODO values.
2. Replace the fixed 0.7 / 0.3 blend with a support-aware backoff, so pairs with weak evidence lean on the category score.
3. Test the surprisingness scores as a correction for article popularity.
4. Add the memory measurements and one scaling plot.
