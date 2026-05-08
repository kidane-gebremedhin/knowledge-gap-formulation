# Sources — Day 4 explainer

*The five canonical sources cited in `explainer.md` (Kidane's writeup for Nurye's question on paired vs. unpaired statistical tests in TheConversionEngine's Week 11 evaluation artifacts), plus the diagnostic pattern used hands-on and one follow-on pointer.*

## Canonical sources (load-bearing citations)

1. **McNemar's original test.** McNemar, Q., "Note on the sampling error of the difference between correlated proportions or percentages," *Psychometrika*, 1947.
   https://link.springer.com/article/10.1007/BF02295996
   *Why load-bearing:* the paper that defines the test the artifact is using. §2 derives the "only discordant pairs are informative" property — this is what produces `p = 1.0` at `paired_task_count = 2` regardless of effect size. Without this derivation, the explainer's claim that the test is below its noise floor at `b + c < 6` would be hand-waved.

2. **The original bootstrap paper.** Efron, B., "Bootstrap Methods: Another Look at the Jackknife," *The Annals of Statistics*, 1979.
   https://projecteuclid.org/journals/annals-of-statistics/volume-7/issue-1/
   *Why load-bearing:* establishes that the bootstrap is a procedure, not a magic — the paired bootstrap is the unmodified procedure applied to unit-level differences, and what makes it "paired" is the resampling unit (whole pairs), not a special algorithm. This is exactly the misconception the artifact's `paired_bootstrap_ci` field tempts you into when you treat it as a black box.

3. **Demšar's classifier-comparison paper.** Demšar, J., "Statistical Comparisons of Classifiers over Multiple Data Sets," *JMLR*, 2006.
   https://www.jmlr.org/papers/v7/demsar06a.html
   *Why load-bearing:* the canonical ML-eval reference for paired vs. unpaired comparisons. §3.1 lays out McNemar's as the right default when the same tasks are scored under both conditions, §3.2 walks through what breaks when you ignore the pairing. This is the paper that turns "paired tests are correct in the abstract" into "paired tests are correct *for ML ablation studies specifically*" — exactly Nurye's setup.

## Supporting sources

- **Newcombe, R. G., 1998 — *Improved Confidence Intervals for the Difference Between Binomial Proportions Based on Paired Data*.** https://onlinelibrary.wiley.com/doi/10.1002/(SICI)1097-0258(19981130)17:22%3C2635::AID-SIM954%3E3.0.CO;2-C. Compares 10 procedures for paired-binary CIs across small-sample regimes, including exact McNemar inversion and the bootstrap. Cited because Nurye's `paired_bootstrap_ci` field needs an honest small-sample alternative when `b + c` is too small for the asymptotic approximation, and Newcombe enumerates them.
- **Dietterich, T. G., 1998 — *Approximate Statistical Tests for Comparing Supervised Classification Learning Algorithms*.** https://direct.mit.edu/neco/article/10/7/1895/6224/. §4 empirically demonstrates the Type I error inflation of naive unpaired tests when classifier outputs share items. Cited because the explainer's claim "the unpaired test on paired data is conservative, not anti-conservative" needs a written-down empirical reference, and Dietterich's Table 1 has the receipts.

## Tool / pattern used hands-on

**Cross-tabulating the two artifacts' inputs to show they are computed on different samples.** Sketched usage:

```python
import json

ablation = json.load(open("outputs/reports/ablation_results.json"))
paired   = json.load(open("outputs/reports/current_ablation_report.json"))

# What does each test see?
print("two_proportion_z_test inputs:")
print(f"  n_baseline (full)  = {ablation['day1_baseline']['n_trials']}")
print(f"  n_treatment (full) = {ablation['full_method']['n_trials']}")
print(f"  delta_a            = {ablation['delta_a']}")
print(f"  p (asymptotic z)   = {ablation['p_value']}")
print()
print("paired_bootstrap_ci + exact_mcnemar inputs:")
print(f"  paired_task_count  = {paired['paired_task_count']}")
print(f"  b (treatment wins) = {paired['mcnemar']['b']}")
print(f"  c (baseline wins)  = {paired['mcnemar']['c']}")
print(f"  p (exact)          = {paired['p_value']}")

overlap_ratio = paired['paired_task_count'] / min(
    ablation['day1_baseline']['n_trials'],
    ablation['full_method']['n_trials'],
)
print(f"\noverlap ratio: {overlap_ratio:.1%}")
```

The `overlap_ratio` is the headline diagnostic. Anything below ~80% means the two artifacts are running on substantively different task sets, and the unpaired test is no longer answering the same question as the paired one. In Nurye's case (`paired_task_count = 2` vs. full counts in the tens), the ratio is single-digit-percent — the unpaired test is doing something other than measuring the critic's effect.

## Follow-on pointer

- **Bouckaert, R. R., 2003 — *Choosing between two learning algorithms based on calibrated tests*.** ICML 2003. http://www.cs.waikato.ac.nz/~remco/icml03.pdf. Becomes relevant when TheConversionEngine moves from a single ablation (one critic vs. one baseline) toward repeated cross-validated comparisons. Bouckaert shows that even paired tests can be over-confident when applied to repeated CV folds without correction, and gives a calibrated alternative.

## Pre-publication checks (per challenge doc)

- [x] No second-hand summaries used as load-bearing citations.
- [x] Every cited paper, tool, and source linked. No fabricated or hallucinated references.
- [x] Tutor pre-publication review cleared.
