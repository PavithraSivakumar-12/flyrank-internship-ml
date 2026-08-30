# FlyRank Capstone — Refresh-Priority Model

**Author:** Pavithra Sivakumar  
**Lane:** Machine Learning  
**Repo:** https://github.com/PavithraSivakumar-12/flyrank-internship-ml  
**Date:** 2026-08-30

## 0. Abstract

This capstone asks which pages from a client's existing content should be prioritized for a refresh review. The analysis uses the local FlyRank content-refresh dataset containing 30,000 rows across 32 pseudonymous clients. A sanctioned `is_declining_label` target was evaluated using leakage-aware features, a transparent rule-based baseline, and Logistic Regression, Decision Tree, and Random Forest models under a client-grouped train/test split. Logistic Regression achieved the strongest held-out-client precision@50 of 0.70 and ROC AUC of 0.611, compared with 0.30 precision@50 for the baseline rule on the same held-out clients. The resulting ranked queue is intended as decision support for editors who need to prioritize pages for human refresh review, not as an automated claim that a page will definitely improve after refreshing.

## 1. Problem framing

The decision supported by this project is:

> Among a client's existing content, which pages should an editor prioritize for a refresh review?

The unit of analysis is an individual content page.

The model produces a decline-risk score for each page and uses that score to create a ranked action queue. The practical action is for a FlyRank editor to review higher-ranked pages first and decide whether a content refresh is appropriate.

A wrong call has two main costs:

- A false positive may cause an editor to spend time reviewing content that does not actually require a refresh.
- A false negative may cause a genuinely declining page to be overlooked.

ML is useful because the dataset contains multiple signals about content age, visibility, engagement, search demand, and other page characteristics. A model can combine these signals into a consistent ranking rather than relying only on one manual rule.

The output is therefore intended as decision support rather than an autonomous publishing or refresh system.

## 2. Data safety

The capstone uses the repository's local starter dataset:

`data/raw/content_refresh_anonymized.csv`

The dataset contains 30,000 rows and 32 pseudonymous clients.

The target is:

`is_declining_label = (trend_direction == "down")`

The following types of information were deliberately excluded from the model:

- `trend_direction` because it directly defines the target.
- `trend_pct` and other label-derived fields because they contain information about the outcome being predicted.
- Pseudonymous identifiers such as `client_id` and `content_id` as model features.
- Provider/model metadata and identifiers that do not represent page-level predictive information.
- Fields whose measurement window overlaps the target's last30/previous30 comparison where appropriate, because they create a potential leakage risk.
- Bucketed or restated variables that duplicate information already represented by included features.

`client_id` was used only for grouped validation. It was not used as a predictive feature.

The final feature set contains numerical and categorical page-level variables including search demand, content size, 90-day visibility and engagement measures, content age, freshness, CTR, average position, engagement rate, scroll rate, AI-traffic percentage, and selected categorical page attributes.

The notebook also explicitly discloses a partial-overlap risk: `log_impressions_90d` and `log_clicks_90d` were among the strongest model features and are temporally close to the target construction. Their importance should therefore not be interpreted as proof of independent causal or predictive power.

No client-identifying details are intentionally included in the capstone outputs. Client identifiers appearing in the anonymized dataset are treated as pseudonymous grouping identifiers rather than human-readable client information.

## 3. Baseline

The transparent baseline flags a page when all three conditions are met:

1. The page has not been updated for at least 90 days.
2. The page had at least 300 impressions during the 90-day window.
3. The average position is worse than 10, or no measured position is available.

The baseline score additionally uses 90-day impressions as a simple ranking signal among flagged pages.

This is a fair comparison because it represents a simple rule that an editor could apply manually without fitting a statistical model.

On the full local dataset:

- Base rate of declining pages: 54.21%
- Baseline flagged rows: 4,294
- Baseline accuracy: 48.93%
- Baseline precision@50: 44.00%

The baseline precision@50 is below the 54.21% base rate. Therefore, simply applying the stale + visible + weak-position rule did not outperform the base-rate reference on that metric.

On the held-out client test split, the baseline achieved:

- Precision@50: 0.30
- Precision@200: 0.38
- Recall: 0.047
- Precision: 0.385
- F1: 0.084
- Accuracy: 0.475

This low baseline establishes a deliberately transparent reference point rather than hiding an unfavorable result.

## 4. Model / analysis

Three supervised classification models were evaluated:

- Logistic Regression
- Decision Tree
- Random Forest

The models were selected because they provide different levels of interpretability and flexibility.

Logistic Regression provides a relatively interpretable linear model. The Decision Tree provides human-readable decision structure. Random Forest provides a more flexible nonlinear comparison.

The target is the sanctioned dataset label:

`is_declining_label = (trend_direction == "down")`

The numerical feature set includes:

- `search_volume`
- `competition`
- `cpc`
- `word_count`
- `char_count`
- `log_impressions_90d`
- `log_clicks_90d`
- `log_sessions_90d`
- `log_ai_sessions_90d`
- `days_with_impressions`
- `days_with_sessions`
- `content_age_days`
- `days_since_last_update`
- `ctr`
- `avg_position`
- `engagement_rate`
- `scroll_rate`
- `ai_traffic_pct`
- `has_keyword_data`
- `has_word_count`

Categorical features include:

- `competition_level`
- `content_type`
- `main_intent`
- `age_tier`
- `freshness_tier`
- `word_count_tier`
- `char_count_tier`

Missing values in selected numerical fields were handled explicitly, and categorical variables were one-hot encoded.

Log transformations were applied to several highly skewed traffic variables:

- `impressions_90d`
- `clicks_90d`
- `sessions_90d`
- `ai_sessions_90d`

The final model selected using held-out-client precision@50 was Logistic Regression.

## 5. Evaluation

The evaluation uses a client-grouped split.

There is no client overlap between training and testing:

- Training rows: 23,837
- Testing rows: 6,163
- Training clients: 25
- Testing clients: 7
- Client overlap: 0

This split was chosen because randomly distributing pages from the same client between training and testing could allow client-specific patterns to appear in both sets and produce an overly optimistic evaluation.

The held-out-client results were:

| Model | ROC AUC | Average Precision | Precision@50 | Precision@200 | Recall | Precision | F1 | Accuracy |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| Logistic Regression | 0.611 | 0.604 | 0.70 | 0.705 | 0.628 | 0.590 | 0.608 | 0.587 |
| Decision Tree | 0.612 | 0.585 | 0.50 | 0.550 | 0.542 | 0.606 | 0.572 | 0.586 |
| Random Forest | 0.599 | 0.582 | 0.52 | 0.470 | 0.581 | 0.587 | 0.584 | 0.577 |
| Baseline rule | — | — | 0.30 | 0.380 | 0.047 | 0.385 | 0.084 | 0.475 |
| Majority-class dummy | — | — | — | — | — | — | — | 0.511 |

The majority-class accuracy is approximately 51.1% on the held-out test set, so accuracy alone is not sufficient to describe model quality.

Logistic Regression was selected because it produced the highest precision@50 among the evaluated models:

- Precision@50: 0.70
- Precision@200: 0.705
- ROC AUC: 0.611
- Average Precision: 0.604

The ROC AUC indicates only moderate discrimination rather than highly accurate prediction. The results therefore support useful ranking signal, but not a strong deterministic predictor.

### Error analysis

The selected Logistic Regression model produced:

- False positives: 1,375
- False negatives: 1,171

Several high-confidence false positives had high impression counts and weak average positions. For example, one false positive had 27,334 impressions, an average position of 76.4, and 103 days since the last update. This suggests that the model can interpret high visibility combined with poor ranking as decline risk even when the observed label is not `down`.

The false negatives included very low-impression pages. Examples had only 2, 3, and 6 impressions over the 90-day period. This suggests that the model has less information available for very low-visibility pages and can miss some declines in that part of the dataset.

These errors reinforce that the ranked output should be reviewed by a human editor.

## 6. Interpretation

The strongest model and permutation-importance signals were:

1. `log_impressions_90d`
2. `log_clicks_90d`
3. `content_age_days`
4. `log_sessions_90d`
5. `avg_position`
6. `word_count`
7. `scroll_rate`
8. `char_count`

The agreement between model importance and permutation importance is useful because it provides two different views of which variables contribute to the model.

The most important finding is that recent traffic and visibility variables dominate the model. However, this should be interpreted cautiously because impressions and clicks are temporally close to the construction of the target and therefore carry a disclosed partial-overlap leakage risk.

A cleaner directional signal is content age: older content tended to contribute more strongly to decline-risk scores.

The model also appears to combine visibility volume with ranking and engagement characteristics. This is useful for prioritization, but it does not establish that any one characteristic causes a page to decline.

The negative result is also important: the model does not achieve a very high ROC AUC. The evidence therefore supports moderate ranking/discrimination rather than a highly reliable page-level prediction.

## 7. Recommendation

The main operational recommendation is to use the model as a ranked review queue.

A FlyRank editor could:

1. Start with the highest-ranked pages.
2. Review the page's current ranking, impressions, clicks, freshness, content quality, and search intent.
3. Decide whether a refresh is actually appropriate.
4. Treat the model score as prioritization evidence rather than an automatic refresh decision.
5. Pay particular attention to pages with very low visibility because the error analysis shows that the model can miss some low-signal declines.

The generated output is:

`capstone_ranked_queue_top200.csv`

The queue contains the top 200 pages from the held-out client evaluation and includes a model probability, relevant page metrics, a reason code, and the recommended action of prioritizing the page for refresh review.

Confidence in the ranking is **moderate**, not high. The ROC AUC of approximately 0.61 indicates that the model contains useful signal but substantial uncertainty remains.

The main limitations are:

- The capstone uses the local 30,000-row starter dataset rather than the full FlyRank warehouse.
- The target is observational and does not establish causality.
- The strongest features have a disclosed partial temporal-overlap leakage risk.
- The model was evaluated on one client-grouped split.
- The results should not be interpreted as predicting Google's ranking algorithm.
- A high model score does not mean that refreshing a page will necessarily improve its performance.

## 8. Reproducibility

The notebook uses:

`work/notebooks/capstone_refresh_priority.ipynb`

and the local dataset:

`data/raw/content_refresh_anonymized.csv`

The primary random seed is:

`42`

The main software stack includes:

- Python
- pandas
- NumPy
- scikit-learn

The workflow can be reproduced from a fresh repository clone by installing the required Python dependencies and running the capstone notebook from the repository's notebook environment.

The notebook performs:

1. Dataset loading.
2. Target construction from the sanctioned `is_declining_label` definition.
3. Leakage-aware feature preparation.
4. Baseline construction.
5. Client-grouped train/test splitting.
6. Model training.
7. Evaluation.
8. Feature-importance analysis.
9. Permutation-importance analysis.
10. Error analysis.
11. Ranked queue generation.
12. Summary metric generation.

The generated reproducibility outputs are:

- `work/outputs/capstone_ranked_queue_top200.csv`
- `work/outputs/feature_importance.csv`
- `work/outputs/permutation_importance.csv`
- `work/outputs/capstone_summary.json`

The client-grouped evaluation is checkable from the notebook: the training set contains 25 clients, the test set contains 7 clients, and the client overlap is 0.

The capstone is explicitly scoped to the local starter dataset because the full warehouse was not available in the execution environment.

## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset: [FlyRank](https://flyrank.ai).
