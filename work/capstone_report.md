# Capstone Report — Refresh / Content Opportunity Scoring

- **Author:** Yousaf Atiq
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/Usaf007/flyrankai-ml-internship
- **Date:** August 2026

---

## 0. Abstract

How can search engineering teams accurately detect web pages before they completely lose their organic rankings? This paper presents a machine learning approach using raw traffic and engagement metrics from 240,691 page records in the FlyRank March 2026 warehouse split. I trained a Random Forest Classifier using features such as Search Console impressions, clicks, GA4 sessions, content age, and word count to predict page decay (average position > 10). Evaluated on an 80/20 train-test holdout set, the model achieved a ROC-AUC of 0.7984 compared to 0.5921 for the Week 4 heuristic baseline—representing a 34.83% lift in discrimination accuracy. The final output is an automated, prioritized action queue that enables editorial teams to optimize their content refresh pipeline with high confidence.

## 1. Problem framing

This work supports the editorial decision of **which decaying web pages should be prioritized for content refreshes**. 

* **Unit of Analysis:** A single web page (`content_hash_id`) on a specific reporting date.
* **Output:** A ranked priority queue featuring an explicit decay probability score and action reason code (`REFRESH_CONTENT`).
* **Human Action:** FlyRank editors review the top-ranked pages to apply targeted content updates, structural rewrites, or internal link boosts.
* **Cost of a Wrong Call:** 
  * *False Positives:* Editorial labor and engineering time are wasted rewriting pages that are already performing fine.
  * *False Negatives:* Undetected decaying pages drop off Page 1 of search engine results, leading to lost organic traffic and revenue.

Machine learning improves this process by evaluating non-linear interactions across traffic metrics and content age simultaneously, outperforming fixed single-variable rules.

## 2. Data safety

* **Warehouse Release:** FlyRank Internship Warehouse (March 2026 split).
* **Tables Joined:** `fact_content_daily_performance` joined with `dim_content` on `content_hash_id`.
* **Exclusions & Filtering:** 
  * Explicitly excluded records where `ga4_data_available IS FALSE` to eliminate zero-filled null sessions that corrupt training data.
  * Filtered out zero-impression rows (`gsc_impressions > 0`).
  * Deliberately removed product-derived flag columns (`health_score`, `priority_score`) to prevent label leakage.
* **Privacy Confirmation:** No client-identifying names, proprietary URLs, raw queries, or system credentials are contained in this analysis or repository artifacts.

## 3. Baseline

The benchmark is the transparent **Week 4 Heuristic Action Score**:
Baseline Score = age_days * ln(gsc_impressions + 1)

This heuristic assumes that older pages with high impression volumes decay fastest. Evaluated on the exact same 20% holdout test set as the model, the Baseline achieved a **ROC-AUC of 0.5921**. While slightly better than random guessing (0.50), it fails to capture complex engagement drops.

## 4. Model / analysis

* **Target Proxy Definition:** `target_is_declining` = 1 if `gsc_avg_position > 10` else 0.
* **Model Choice:** `RandomForestClassifier` (`n_estimators=100`, `max_depth=10`, `random_state=42`). Capping tree depth at 10 prevents overfitting and enforces generalization across unseen time windows.
* **Feature List:**
  * `gsc_impressions`: Total Google Search Console impressions.
  * `gsc_clicks`: Total Google Search Console clicks.
  * `ga4_sessions`: Verified Google Analytics 4 sessions.
  * `age_days`: Total days since the content was published.
  * `word_count`: Document length in words.
* **Deliberately Excluded:** `trend_direction` and `trend_pct` (label-derived leakage risks), as well as pseudonymous IDs.

## 5. Evaluation

* **Split Strategy:** Standard 80/20 train-test split on 240,691 valid records.
* **Task Base Rate:** **52.44%** of records naturally met the decay threshold, providing a balanced discrimination task.

### Honest Performance Comparison (Holdout Test Set)

| System | ROC-AUC | Lift over Baseline |
| :--- | :--- | :--- |
| **Week 4 Heuristic Baseline** | 0.5921 | Baseline |
| **Random Forest Classifier** | **0.7984** | **+34.83%** |

* **Error Analysis:** The model occasionally misclassifies high-impression, long-tail pages whose traffic dips during seasonal weekend patterns. Incorporating day-of-week feature interactions in future iterations will resolve these periodic false signals.

## 6. Interpretation

* **Key Finding:** Engagement signals (`ga4_sessions` and `gsc_clicks`) combined with `age_days` drive the model's split decisions far more than raw `word_count`.
* **Surprise / Counter-Intuitive Result:** In alignment with earlier signal audits, content age alone does not linearize decay. Established pages (>365 days) often retain ranking stability due to entrenched authority, whereas newer pages (<180 days) exhibit higher ranking volatility. The Random Forest captures these non-linear boundaries automatically.

## 7. Recommendation

The output yields an actionable playbook saved directly to `work/outputs/capstone_ranked_queue.csv`. 

### Operational Workflow for FlyRank Editors:
1. Open the daily generated ranked queue sorted by `ml_decay_probability`.
2. Filter for high-value targets (e.g., `ml_decay_probability > 0.85` and `gsc_impressions > 500`).
3. Apply the recommended `REFRESH_CONTENT` action by updating out-of-date facts, optimizing heading structures, and bolstering internal links.

*Framing Limit:* This model provides **decision-support metrics**. It does not guarantee algorithmic recovery, and editors must confirm page intent before rewriting.

## 8. Reproducibility

To reproduce all results from a clean repository clone:

1. Environment setup:
    `pip install duckdb pandas numpy scikit-learn`
2. Set Hugging Face Token:
    `export HF_TOKEN="your_huggingface_read_token"`
3. Run the complete capstone notebook:
   Open and execute `work/notebooks/capstone.ipynb` top-to-bottom.
4. **Random Seed:** Set to `42` across all data splitters and estimators.
5. **Generated Artifacts:** Output files `capstone_metrics.csv` and `capstone_ranked_queue.csv` are deterministically written to `work/outputs/`.

## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset hosted at https://flyrank.ai.