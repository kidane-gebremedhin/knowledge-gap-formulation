# How Much Should You Trust a Small Reused Hold-Out Set?

## Peer Question

Kidane Gebremedhin Gidey's question is about a very practical evaluation problem from his Week 11 `sales-agent-evaluation-bench`. He trained a SimPO adapter on 109 preference pairs and evaluated it on a 50-example held-out split. The model reached 94.3% accuracy on training and 86.0% on the held-out set, producing an 8.3-point train–held-out gap. At first glance, that looks like reasonable generalization. But the held-out set is small, and it has also been used repeatedly during iterative development.

Restated plainly, the question is this: when a small held-out partition is used as the main evaluation set throughout development, how should we estimate how trustworthy that held-out accuracy still is as evidence of true generalization? And how should we report the train–held-out gap so readers can distinguish real generalization from sampling noise and adaptive evaluation bias?

## Why This Question Matters

This matters because a single number can look much more solid than it really is. If `MODEL_CARD.md` headlines 86.0% held-out accuracy without explaining the uncertainty around 43 correct predictions out of 50, and without explaining that the same split was reused during development, readers may treat that number as a test-set estimate of out-of-sample performance when it is really closer to a reused validation estimate.

That difference matters for real model decisions. It affects whether you can honestly claim the model generalizes, whether you should add a separate untouched test set now, and whether the current benchmark should be treated as deployment evidence or only as development guidance.

## Short Answer

The right empirical framing is:

1. Treat the 50-example held-out set as a **validation/dev set**, not as a final unbiased test set, if it was consulted repeatedly during development.
2. Report the held-out accuracy with an **explicit finite-sample confidence interval**, not just a point estimate.
3. State clearly that this interval captures **sampling uncertainty conditional on this split**, but does **not** capture optimism introduced by adaptive reuse of the same held-out set.
4. Treat the 8.3-point train–held-out gap as an **observed train–dev gap**, not as definitive proof of true generalization.
5. For a stronger final claim, freeze the modeling pipeline and evaluate once on a genuinely untouched test set, ideally after the V0.2 dataset expansion gives enough data to afford that split.

So the right answer is not "throw away the current held-out number." It is "downgrade its status, quantify its uncertainty honestly, and avoid presenting it as a final generalization estimate."

## Literature Review

Three primary sources are especially useful here.

The first is Wilson's classic 1927 paper introducing what is now called the Wilson score interval for binomial proportions. This matters because held-out accuracy on 50 examples is a binomial proportion, and the common normal-approximation Wald interval is unreliable at small sample sizes. A proportion like 43/50 should be reported with an interval designed for finite-sample binomial uncertainty, not just a point estimate and a rough standard error. Source: [Wilson, "Probable Inference, the Law of Succession, and Statistical Inference"](https://doi.org/10.1080/01621459.1927.10502953).

The second is Dwork et al.'s work on adaptive data analysis and the reusable holdout. The important lesson is that a hold-out set is not magically immune to bias if it is queried repeatedly during model iteration. Once model changes are informed by the same held-out results, the reported performance can become optimistic relative to a truly untouched evaluation. Source: [Dwork et al., "The reusable holdout: Preserving validity in adaptive data analysis"](https://research.ibm.com/publications/the-reusable-holdout-preserving-validity-in-adaptive-data-analysis) and the broader treatment [Dwork et al., "Preserving Statistical Validity in Adaptive Data Analysis"](https://dwork.seas.harvard.edu/publications/preserving-statistical-validity-adaptive-data-analysis).

The third is Varma and Simon's paper on bias in error estimation when cross-validation is used for model selection. Although Kidane's setup is not literally the same as nested cross-validation, the methodological point carries over: once an evaluation set is used to guide modeling decisions, its error estimate becomes biased for final performance assessment. Source: [Varma and Simon, "Bias in error estimation when using cross-validation for model selection"](https://bmcbioinformatics.biomedcentral.com/articles/10.1186/1471-2105-7-91).

Cawley and Talbot reinforce the same general point from the broader model-selection perspective: even a nearly unbiased model-selection criterion can still induce substantial selection bias if it is noisy and repeatedly optimized against. Source: [Cawley and Talbot, "On Over-fitting in Model Selection and Subsequent Selection Bias in Performance Evaluation"](https://jmlr.org/papers/v11/cawley10a.html).

Together, these sources support the two main pieces of the answer:

1. the 50-example held-out accuracy has non-trivial finite-sample uncertainty
2. repeated development against that held-out split weakens its interpretation as a final generalization estimate

## Core Mechanism

There are really two different uncertainty problems in Kidane's number, and they should not be collapsed into one.

### 1. Sampling uncertainty

If a model gets 43 out of 50 held-out examples correct, the point estimate is 86.0%. But with only 50 examples, that number has noticeable finite-sample noise. A different set of 50 held-out examples from the same underlying task distribution could easily give a different observed accuracy.

This is the classical binomial-proportion problem. The held-out accuracy is not a fixed property we observed perfectly; it is a sample estimate of an unknown population accuracy.

### 2. Adaptive reuse bias

Now add the second problem: the held-out split is not being used once at the very end. It is being used during iterative development to compare choices, tune behavior, and decide what to keep. That means the development process is indirectly fitting to the held-out set.

Even if nobody explicitly "trains on the held-out labels," the workflow is still adaptive. Model changes that happen to do well on that split are more likely to survive. Over time, that can make the held-out estimate optimistic relative to performance on a truly untouched sample.

So:

- the confidence interval tells you about **sampling noise**
- the adaptive-reuse problem adds **selection bias**

Those are different error sources. A standard confidence interval only addresses the first one.

## Case Study From Our Work

Kidane's exact setup makes this concrete:

- training accuracy: 94.3% on 109 examples
- held-out accuracy: 86.0% on 50 examples
- observed gap: 8.3 points

At first glance, this looks like a plausible generalization story: training is higher, held-out is lower, but not catastrophically lower.

But there are three reasons this should be interpreted cautiously.

First, the held-out set is small. Fifty examples is enough to be useful for development, but not enough to make a narrow headline claim without uncertainty.

Second, the training accuracy is in-sample. It is expected to be optimistic relative to held-out performance. So the train–held-out gap is not a symmetric comparison between two equally reliable estimates of the same target.

Third, if the same 50-example split shaped iterative modeling decisions, then the 86.0% is no longer a clean one-shot test estimate.

That means the number is still valuable, but it should be described more carefully: it is evidence from a reused validation partition, not final proof of generalization.

## Concrete Demonstration

Let us compute the uncertainty on the held-out number directly.

Kidane's held-out accuracy is:

- `43 / 50 = 86.0%`

Using a Wilson 95% interval, that corresponds to approximately:

- **73.8% to 93.0%**

So even before talking about adaptive reuse, the finite-sample uncertainty is already substantial.

The training number is:

- `103 / 109 = 94.5%`

Its Wilson 95% interval is approximately:

- **88.5% to 97.5%**

Now look at the observed train–held-out gap:

- `94.5% - 86.0% ≈ 8.5 points`

If you do a rough independent-binomial uncertainty calculation for that gap, the 95% uncertainty band is approximately:

- **-2.0 points to +19.0 points**

That rough calculation is not the final answer, because training accuracy is in-sample and the held-out set has been reused adaptively. But it does make one thing clear: the observed 8.3-point gap is not precise enough to support strong claims on its own.

That is the right way to interpret the current numbers:

- the model may generalize reasonably
- but the evidence is still coarse
- and the held-out estimate is not clean enough to be treated as final

## Adjacent Concepts

### Validation set versus test set

This question is partly about terminology discipline. A split that actively guides model iteration is a validation or dev set, even if it was originally created as a held-out partition. A test set is supposed to remain untouched until the pipeline is frozen.

### Selection bias from model iteration

This is closely related to overfitting the model-selection criterion. Even if no gradient step is taken on the held-out examples, repeated selection pressure can still bias the estimate upward.

### Resolution versus credibility

With only 159 total pairs, there is a real tradeoff between using more data for training and keeping enough untouched data for credible final evaluation. That is why waiting for the V0.2 dataset expansion can be a rational choice, but only if the current metric is reported honestly as a reused validation estimate.

## Limitations and Tradeoffs

There are real tradeoffs here.

First, creating a separate untouched test partition now would reduce development resolution. With only 50 held-out examples already, carving out another final test set may make the dev signal too noisy to guide iteration well.

Second, not creating a test set now means the current held-out number should not be over-sold. The price of keeping more development resolution is weaker final evidence.

Third, nested cross-validation or repeated resampling can help with development-stage estimation, but they are more complex operationally and may still be harder to explain in a model card than a simple train/dev/test story.

Fourth, no statistical interval will "fix" adaptive reuse bias by itself. Confidence intervals quantify random sampling variation, not the hidden optimism created by development decisions.

## What I Would Change in the Portfolio After Learning This

I would make three concrete changes to Kidane's evaluation reporting.

First, I would revise `MODEL_CARD.md` so the headline is not simply "86.0% held-out accuracy." I would report it as:

- **Validation accuracy: 86.0% (43/50; Wilson 95% CI 73.8%–93.0%)**

and I would explicitly note that this split was reused during development.

Second, I would relabel the 8.3-point train–held-out gap as an **observed train–validation gap**, not a definitive generalization gap. I would say that it is consistent with reasonable generalization, but too noisy and development-exposed to serve as strong standalone evidence.

Third, I would plan the next empirical step around the V0.2 expansion: keep the current split for development now, but reserve a genuinely untouched test partition once enough additional labeled pairs exist to make that affordable.

## Sources

- Wilson, E. B. (1927), "Probable Inference, the Law of Succession, and Statistical Inference"  
  https://doi.org/10.1080/01621459.1927.10502953
- Dwork et al., "The reusable holdout: Preserving validity in adaptive data analysis"  
  https://research.ibm.com/publications/the-reusable-holdout-preserving-validity-in-adaptive-data-analysis
- Dwork et al., "Preserving Statistical Validity in Adaptive Data Analysis"  
  https://dwork.seas.harvard.edu/publications/preserving-statistical-validity-adaptive-data-analysis
- Varma and Simon, "Bias in error estimation when using cross-validation for model selection"  
  https://bmcbioinformatics.biomedcentral.com/articles/10.1186/1471-2105-7-91
- Cawley and Talbot, "On Over-fitting in Model Selection and Subsequent Selection Bias in Performance Evaluation"  
  https://jmlr.org/papers/v11/cawley10a.html

Direct answer to the question: when a small held-out partition has been reused throughout development, the correct empirical procedure is to treat it as a validation set, report its finite-sample uncertainty explicitly, and avoid presenting it as a final unbiased measure of generalization. The train–held-out gap should be reported as an observed train–validation gap with two caveats: sampling noise from the small held-out size, and optimism from adaptive reuse. A separate untouched test set is the cleanest fix, but if the dataset is still too small, the honest interim choice is to keep using the held-out split for development while clearly downgrading the strength of the generalization claim until the V0.2 expansion makes a proper final test set practical.
