# Tweet thread — Day 4

*Compressed from `explainer.md`. 6 tweets. Each stands alone for a reader who never clicks through.*

---

**1/6**

A teammate asked: "My ablation report has two p-values for the same comparison. The unpaired two-proportion z-test says +16 lift, p = 0.018. The paired McNemar says p = 1.0 with paired_task_count = 2. Which is right?"

Neither. They're answering different questions on different data.

---

**2/6**

The unpaired test models the two arms as independent samples from two populations. Variance: `var(p̂₂) + var(p̂₁)`.

The paired test models the same units measured twice. Variance: `var(p̂₂) + var(p̂₁) − 2·cov(p̂₂, p̂₁)`.

Same units → positive cov → smaller paired variance → **more** power.

Using the unpaired test on paired data is conservative, not anti-conservative.

---

**3/6**

So why does the paired test give p = 1.0 when the unpaired gives p = 0.018?

It's not the same data.

The unpaired test rolled up the full counts each arm saw — possibly tens of trials each. The paired test restricted to the **2 task IDs the arms shared**.

Different denominators, different units, different question.

---

**4/6**

p = 1.0 at paired_task_count = 2 doesn't mean "no effect." It means **the test cannot detect anything** at that N.

Math: McNemar's exact at b + c = 2 has a minimum two-sided p of 0.5. To reach p < 0.05 you need b + c ≥ 6 discordant pairs. Below that, the test is below its own noise floor.

---

**5/6**

The +16 lift in the unpaired test isn't measuring the system effect either. It's measuring:

system_effect + (avg_task_difficulty_in_arm_B − avg_task_difficulty_in_arm_A)

Without overlap, the two are not separately identifiable.

If your two arms ran on different task sets, the unpaired test's "lift" is a confound, not an estimate.

---

**6/6**

Decision tree for any future eval:

- Same tasks under both arms → paired test (McNemar exact for binary, paired bootstrap for everything else).
- Partial overlap → restrict to overlap, run paired, report N. Don't fall back to unpaired on full counts — it answers a different question.
- No overlap → you cannot identify the system effect. Fix the data, not the test.

Sources:
• McNemar (1947) https://link.springer.com/article/10.1007/BF02295996
• Demšar (2006) https://www.jmlr.org/papers/v7/demsar06a.html
• Dietterich (1998) https://direct.mit.edu/neco/article/10/7/1895/6224/
