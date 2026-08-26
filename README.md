# Landing Page A/B Test Analysis

Statistical analysis of an A/B test run on an e-commerce landing page, evaluating whether a new page design outperforms the existing one in converting visitors into paying users. The result is validated three independent ways — bootstrap simulation, a two-sample z-test, and logistic regression — all of which agree on the same conclusion.

## Business Question

A company redesigned its landing page and ran an experiment splitting visitors between the old page (`control`) and the new page (`treatment`). The goal of this analysis is to answer, from the data alone, whether the company should:

1. Implement the new page,
2. Keep the old page, or
3. Run the experiment longer before deciding.

## Dataset

| File | Description |
|---|---|
| `ab_data.csv` | 294,478 rows of visit-level data: `user_id`, `timestamp`, `group` (`control`/`treatment`), `landing_page` (`old_page`/`new_page`), `converted` (0/1) |
| `countries.csv` | Maps each `user_id` to a country (`US`, `UK`, `CA`), used in the regression extension |

## Data Cleaning

Before any analysis, the raw data required cleaning:

- **Mismatched assignments**: 3,893 rows had a `group`/`landing_page` combination that didn't line up (e.g. a `control` user shown the `new_page`). Since it's impossible to know which page these users actually saw, these rows were dropped, leaving 290,585 rows.
- **Duplicate user**: one `user_id` (`773192`) appeared twice with different timestamps. The duplicate was removed, leaving one row per unique user (290,584 users).

## Methodology

The analysis is split into three independent approaches, each testing the same underlying question — does the new page convert better than the old page — using a different statistical tool.

### 1. Simulation-Based Hypothesis Testing (Bootstrap)

- **H₀**: p_new ≤ p_old (the new page converts the same or worse than the old page)
- **H₁**: p_new > p_old (the new page converts better)

Under the null, both groups are assumed to share a single population conversion rate. This shared rate was used to simulate 10,000 pairs of samples (matching the real group sizes), computing the difference in simulated conversion rates each time to build a null sampling distribution. The actual observed difference was then compared against this distribution.

- **Observed difference** (treatment − control): **-0.00158**
- **p-value** (proportion of simulated differences exceeding the observed difference): **≈ 0.910**

Since the p-value is far above any conventional significance threshold (e.g. 0.05), we **fail to reject the null hypothesis** — there's no evidence the new page performs better.

### 2. Two-Sample Z-Test

The same hypothesis was tested using the closed-form `statsmodels.stats.proportions_ztest`, comparing the two conversion counts directly rather than simulating.

- **Conversions**: 17,489 (old page, n=145,274) vs. 17,264 (new page, n=145,310)
- **z-score**: 1.311
- **p-value**: 0.905

This closely matches the simulation-based result, confirming the conclusion: **fail to reject H₀**.

### 3. Logistic Regression

As a third, independent check, conversion was modeled directly as a function of which page a user saw, using logistic regression (`statsmodels.Logit`).

- **Base model** (`converted ~ ab_page`): p-value for `ab_page` = 0.190 → not statistically significant at α = 0.05.
- **Extended model** (adding country: `US`, `UK`, and page × country interaction terms): none of the added terms reached significance either, and the pseudo R² remained ≈ 0.
- A train/test logistic regression classifier (80/20 split) was also fit to check predictive power — it produced **0.0 precision and recall**, confirming the model has no discriminative power. This is expected given the near-zero relationship between page version and conversion; it does not change the conclusion above.

**Note on hypothesis framing**: the simulation and z-test use a one-sided hypothesis (new page ≥ old page), while the regression naturally tests a two-sided hypothesis (pages differ at all). Both framings converge on the same non-significant result.

## Results

| Method | Statistic | p-value | Conclusion |
|---|---|---|---|
| Bootstrap simulation (10,000 iterations) | diff = -0.00158 | ≈ 0.910 | Fail to reject H₀ |
| Two-sample z-test | z = 1.311 | ≈ 0.905 | Fail to reject H₀ |
| Logistic regression | — | 0.190 | Not significant |

All three methods agree: **there is no statistically significant difference in conversion rate between the new and old landing pages**, and adding country as a covariate does not change this. Precision/recall of 0 on the classification task further confirms that page version carries no predictive signal for conversion.

## Business Recommendation

Based on this analysis, the company should **not roll out the new landing page**. The observed difference in conversion rates is small (-0.16 percentage points, favoring the old page) and is well within the range expected by chance. Launching the new page would carry the cost and risk of a redesign without any demonstrated lift in conversions. If the company wants to keep exploring page variants, the recommendation is to design a follow-up experiment with a clearer expected effect size and, if needed, a longer run to detect smaller effects with adequate power.

## Tech Stack

- **Python**: pandas, NumPy
- **Statistics**: statsmodels (`proportions_ztest`, `Logit`), bootstrap resampling
- **Machine Learning**: scikit-learn (`LogisticRegression`, train/test split, precision/recall)
- **Visualization**: Matplotlib

## Repository Structure

```
.
├── Analyze_ab_test_results_notebook.ipynb   # Full analysis notebook
├── ab_data.csv                              # Visit-level experiment data
├── countries.csv                            # User-to-country mapping
└── README.md
```

## How to Run

```bash
pip install pandas numpy matplotlib statsmodels scikit-learn jupyter
jupyter notebook Analyze_ab_test_results_notebook.ipynb
```

## Key Takeaway

Statistical significance testing exists precisely to prevent decisions like "the new page converted slightly worse, so let's scrap it" or "let's ship it because it feels newer" — small, noisy differences are common in real data and don't imply a real effect. Here, three independent methods (simulation, closed-form z-test, and regression) converge on the same answer, which is what gives the recommendation confidence: **the data does not support switching to the new page.**
