# Capstone Report — Content Refresh Opportunity Scoring

- **Author:** Hasham Hassan
- **Lane:** Machine Learning
- **Repo:** https://github.com/HashamHassan-01/flyrank-ml-internship-hasham
- **Date:** 2026-08-11

> Copy this file to `work/capstone_report.md` and fill it in as you build. Sections 1–8
> mirror the Pass / Needs-Work rubric axes, so nothing here is optional. Sections 0 and 9
> are **paper sections**: your deployed research paper must carry both, and they're here so
> you never rebuild them from memory at ship time.

## 0. Abstract

This study asks whether page-level search and content signals can support a consistent ranking of pages for content-refresh review. The analysis uses an anonymized dataset of 30,000 content pages and evaluates search volume, content age, average search position, and CTR. A transparent Opportunity Score was used as the baseline, and a Random Forest Regressor was evaluated using a client-grouped train/test split with zero client overlap. The model closely reproduced the constructed scoring rule, while the signal audit found directional evidence for combining content age, CTR, and search position and weaker evidence for search volume as a standalone signal. The resulting output is intended as decision-support for human editors to prioritize pages for review, not as proof that a refresh will improve future performance.

## 1. Problem framing

The goal is to support content-refresh prioritization: identify pages that are stronger candidates for human review and potential refresh.

The unit of analysis is an individual content page. The output is an Opportunity Score and ranked action queue that helps editors decide which pages to review first.

The human action is not automatic publishing or deletion. Instead, an editor reviews the ranked pages and decides whether a refresh is appropriate.

A wrong call can waste editorial effort on a page that does not need attention, while failing to prioritize a genuinely weak or aging page can delay a potentially useful refresh.

Machine learning is useful here because it can combine multiple page-level signals into a consistent ranking that helps editors prioritize a large content inventory. However, the resulting score is a decision-support signal rather than proof that a page will improve after a refresh.

## 2. Data safety

The analysis used the anonymized content-refresh dataset containing 30,000 rows and 44 columns. The required modeling fields were `content_id`, `client_id`, `search_volume`, `content_age_days`, `avg_position`, and `ctr`.

For modeling, rows missing required numeric inputs were excluded. This left 27,076 rows for modeling, with 31 unique clients; 2,924 rows were removed during preparation.

The analysis deliberately excluded `content_id` and `client_id` from the model features. `client_id` was used only to create a grouped train/test split so that pages from the same client could not appear in both sets. This produced 24 training clients and 7 test clients with zero client overlap.

Potential label-derived fields such as `trend_direction` and `trend_pct` were not used as model features because they could introduce leakage. The model used only the selected page-level signals: `search_volume`, `content_age_days`, `avg_position`, and `ctr`.

The dataset and analysis are anonymized. No client names, private URLs, or private search queries are included in the report or public-facing artifacts.

## 3. Baseline

The baseline was a transparent Opportunity Score constructed from four page-level signals: search volume, content age, average search position, and CTR.

The score was defined as:

- 40% search-volume signal
- 25% average-position signal
- 20% content-age signal
- 15% CTR signal

The baseline provides a simple and interpretable reference because the same page-level inputs used for prioritization are combined with fixed weights rather than learned from the data.

The Random Forest model was evaluated against this baseline using the same grouped client-level test split and the same error metrics. This makes the comparison consistent and shows whether the learned model reproduces or improves on the transparent scoring rule.

An important limitation is that the baseline score is a constructed proxy rather than an independently observed business outcome. Therefore, strong agreement between the model and baseline does not demonstrate that either one predicts the real-world success of a content refresh.

## 4. Model / analysis

The modeling approach used a Random Forest Regressor to learn the relationship between the selected page-level signals and the Opportunity Score proxy.

The exact model features were:

- `search_volume`
- `content_age_days`
- `avg_position`
- `ctr`

`content_id` and `client_id` were intentionally left out of the feature set. They identify pages or clients but do not provide a meaningful generalizable content signal, and using them could encourage memorization of client-specific patterns.

The target was the Opportunity Score proxy constructed from the same four page-level signals using the transparent baseline weighting described above.

The Random Forest used 100 trees with `random_state=42`. The grouped client-level validation was used to reduce the risk of client-specific patterns appearing in both training and test data.

The purpose of the model was therefore to reproduce and rank the constructed Opportunity Score consistently across unseen clients, not to claim prediction of an independent business outcome.

## 5. Evaluation

The Random Forest model was evaluated against a simple mean baseline using MAE, RMSE, and R².

| Model         |      MAE |     RMSE |        R² |
| ------------- | -------: | -------: | --------: |
| Mean baseline | 0.052463 | 0.059494 | -0.013312 |
| Random Forest | 0.002149 | 0.006581 |  0.987601 |

The Random Forest performs substantially better than the mean baseline. Its MAE is **0.002149** compared with **0.052463** for the baseline, while its RMSE is **0.006581** compared with **0.059494**. The Random Forest also achieves an **R² of 0.987601**, whereas the mean baseline has an R² of **-0.013312**.

These results show that the model reproduces the target score very closely on the evaluation split. However, the target score is calculated from the same input features used by the model, so this high performance should not be interpreted as proof that the model predicts an independent real-world outcome. It primarily demonstrates that the Random Forest can reproduce the existing scoring formula.

Therefore, the model is suitable as a scoring/reproduction mechanism for the current workflow, but further validation with an independent business outcome would be required before treating it as a predictive model for real-world content performance.


## 6. Interpretation

The Random Forest model reproduced the constructed Opportunity Score closely because the target was derived directly from the same four page-level signals used as model inputs.

The main signals examined in the analysis were search volume, average position, content age, and CTR. The earlier signal audit found that the relationship between content age and CTR was weak (`Spearman rho = -0.0585`, MIXED), while CTR and average position showed a stronger directional relationship (`Spearman rho = -0.2342`, CONFIRMED after excluding `avg_position = 0`). Search volume and CTR also showed a weak relationship (`Spearman rho = -0.0756`, MIXED).

The flag-linked test provided directional support for using content age as a review signal: the oldest 25% of pages had a median CTR of 0.05 compared with 0.08 for the remaining pages, a measured difference of -0.03.

A key negative result is that these relationships do not prove that refreshing older pages will improve CTR or search performance. The model's strong reproduction of the baseline should therefore be interpreted as successful replication of the scoring rule, not as evidence of independent real-world predictive power.

## 7. Recommendation

The output should be used as a ranked decision-support queue for human content review rather than as an automatic refresh rule.

### Ranked actions

1. **Review older pages with lower CTR first.**  
   The oldest 25% of pages had a measured median CTR of 0.05 compared with 0.08 for the remaining pages. This provides directional evidence for using content age together with CTR as a review signal.

2. **Review lower-CTR pages with weaker search position as a combined signal.**  
   After excluding `avg_position = 0` records because they represent no valid position data, CTR and average position showed a Spearman correlation of -0.2342. This relationship can help prioritize pages for inspection, but it does not establish causation.

3. **Use search volume as a secondary signal.**  
   Search volume and CTR showed a weak Spearman relationship of -0.0756, so search volume should not be used as a standalone refresh rule.

A FlyRank editor could use the ranked queue to decide which pages deserve attention first, then inspect the page and its context before deciding whether a refresh is appropriate.

Confidence is directional rather than predictive. The evidence supports prioritization and review, but the current analysis cannot establish that refreshing a selected page will improve its future performance. An independent post-refresh outcome would be needed to validate that claim.

## 8. Reproducibility

The analysis is contained in the notebooks committed under `work/notebooks/` in the project repository.

The main capstone notebook is:

`work/notebooks/capstone.ipynb`

The signal-audit analysis is:

`work/notebooks/w04_signal_audit.ipynb`

The project can be inspected or rerun from the repository after cloning it and opening the notebooks in Google Colab or a compatible Jupyter environment.

The modeling workflow uses `random_state=42` for the Random Forest and the grouped train/test split. The evaluation uses a client-level split with zero client overlap between training and test sets.

The modeling dataset and evaluation outputs are generated from the anonymized starter dataset available in the repository. The analysis does not use client names, private queries, or private URLs.

This work does not claim a sealed blind evaluation. The target is a constructed Opportunity Score proxy, so the results should be treated as reproducible analysis of that scoring rule rather than independent validation of real-world refresh outcomes.

## 9. Acknowledgments & data credit

Built on the [FlyRank ML Internship dataset](https://flyrank.ai).

> **Claims checklist before submitting:** observed / measured / directional / decision-support
> **Metrics vs. base rate:** report your task's base rate (majority-class %) next to any
> precision@K or accuracy — a high score can just be a high base rate. AUC / lift over
> baseline are the honest discrimination numbers.
> language everywhere · no causal claims without an experiment or causal design · no
> "predicted Google's algorithm" · no client-identifying details · numbers in this report
> match a fresh re-run.
