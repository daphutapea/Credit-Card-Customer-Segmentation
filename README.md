# Credit Card Customer Segmentation

Behavioral segmentation of 4,475 credit card holders into three actionable customer
personas, using K-Means over PCA-reduced usage features. No demographic data is used:
every segment is defined purely by what customers do with the card.

**Stack:** Python, scikit-learn, pandas, Google BigQuery
**Task:** Unsupervised learning (clustering)
**Status:** Complete, with serialized artifacts for inference

---

## The problem

A bank's marketing team sends one credit card offer to the entire portfolio. This project
asks whether that portfolio is actually one audience.

It is not. Average spend per customer is $1,003 while the *median* customer spends $362,
because the spend distribution is heavily right-skewed (skewness 9.0). Planning around the
average means planning for a customer who does not exist in the data.

The goal is to split the portfolio into behaviorally distinct groups and attach a revenue
engine, a risk level, and a concrete next action to each one.

---

## Results

Three clusters over 4,001 accounts (after outlier filtering).

| Cluster | Persona | Share | Defining behavior | Recommended action |
| --- | --- | ---: | --- | --- |
| 0 | **Revolvers** | 42% (1,675) | Carries $2,096 balance, $1,417 cash advance, pays in full only 2% of months | Balance transfer and lower-APR loan offers; financial-health nudges |
| 1 | **Active Transactors** | 41% (1,630) | $1,183 purchases across 22 transactions, buys in 87% of months, pays in full 22% | Reward-tier upgrade, premium card upsell, partner-merchant offers |
| 2 | **Dormant** | 17% (696) | $121 balance, buys in only 28% of months, but pays in full 24% | Reactivation: cashback on first N transactions, fee waiver, win-back |

Mean values per cluster, on the original (unscaled) features:

| Feature | Cluster 0 | Cluster 1 | Cluster 2 |
| --- | ---: | ---: | ---: |
| `BALANCE` | 2,095.59 | 963.63 | 121.28 |
| `PURCHASES` | 220.51 | 1,182.76 | 341.11 |
| `CASH_ADVANCE` | 1,417.24 | 197.73 | 309.93 |
| `PURCHASES_FREQUENCY` | 0.14 | 0.87 | 0.28 |
| `CASH_ADVANCE_FREQUENCY` | 0.23 | 0.04 | 0.03 |
| `PURCHASES_TRX` | 2.53 | 21.74 | 4.17 |
| `CREDIT_LIMIT` | 3,925.94 | 4,129.44 | 3,462.04 |
| `PRC_FULL_PAYMENT` | 0.02 | 0.22 | 0.24 |

**The headline finding:** all three groups carry roughly the same credit limit
($3,462 to $4,129). The bank gave them an identical product and they did completely
different things with it. Segmentation is the only way to see that.

---

## Exploratory findings

**1. Account tenure predicts repayment, but not balance.**

Financial features here are heavily right-skewed (`PURCHASES` skew 9.0, `PAYMENTS` 6.0),
which violates Pearson's normality assumption. Switching to Spearman roughly doubled the
measured relationship:

| Pair | Pearson r | Spearman rho | Reading |
| --- | ---: | ---: | --- |
| `TENURE` vs `PAYMENTS` | 0.098 | **0.212** | Weak positive, the strongest of the three |
| `TENURE` vs `PURCHASES` | 0.086 | **0.139** | Weak positive |
| `TENURE` vs `BALANCE` | 0.073 | **0.070** | Negligible |

Long-tenure customers show 2 to 3 times the median purchases, balance and payments of
6-month customers. But `BALANCE` is effectively unrelated to tenure, so tenure must not be
used as a proxy for credit health.

*Caveat:* 85% of accounts sit at `TENURE` = 12, so the short-tenure end is thinly sampled.

**2. Credit limit is associated with purchase frequency, but weakly.**

| Credit limit tercile | Range | n | Mean `PURCHASES_FREQUENCY` |
| --- | --- | ---: | ---: |
| Low | $150 - $2,000 | 1,519 | 0.445 |
| Medium | $2,050 - $5,000 | 1,502 | 0.475 |
| High | $5,100 - $30,000 | 1,453 | 0.553 |

Kruskal-Wallis confirms the groups genuinely differ (H = 60.28, p = 8.1e-14), and Spearman
on the raw continuous variable gives rho = +0.123. High-limit customers buy about 24% more
often than low-limit ones.

However, credit limit explains only about 1.5% of the variance in purchase frequency. With
4,475 records almost any effect clears the significance bar, so effect size matters more
than the p-value here: a blanket limit increase would not move engagement.

---

## Method

```
Raw extract (4,475 x 18)
  -> drop CUST_ID, keep 17 behavioral features
  -> SimpleImputer(strategy='mean')      159 missing values, 0 duplicates
  -> StandardScaler()                    K-Means is distance-based
  -> IsolationForest(random_state=42)    474 accounts removed (10.6%), 4,001 kept
  -> PCA(n_components=0.95)              17 features -> 11 components, 96.12% variance
  -> KMeans(n_clusters=3, n_init=10)
```

**Why Isolation Forest and not Local Outlier Factor?** Both were run; LOF was used only as
a cross-check (it flagged 236 accounts, 105 shared with Isolation Forest). LOF scores a
point against its immediate neighbours, which risks deleting small but legitimate customer
groups, exactly what segmentation exists to find. Isolation Forest scores each account
against the whole portfolio instead.

**Why mean imputation?** Only 3.5% of rows are affected (158 in `MINIMUM_PAYMENTS`, 1 in
`CREDIT_LIMIT`). Mean imputation preserves the distribution ahead of standard scaling. This
does introduce a mild bias and is listed under limitations below.

### Choosing k

| k | Silhouette |
| ---: | ---: |
| 2 | 0.2318 |
| **3** | **0.2390** |
| 4 | 0.2190 |
| 5 | 0.2341 |
| 6 | 0.2525 |
| 7 | 0.2523 |
| 8 | 0.2574 |

Silhouette does not peak at k=3, and that is stated deliberately. The entire curve spans
0.038, so silhouette is not discriminating between these options. k=3 was chosen on four
independent signals:

1. **Elbow** - the largest single drop in WCSS is k=2 to k=3.
2. **Silhouette** - 0.239 at k=3; the best score anywhere on the curve beats it by 0.018.
3. **Davies-Bouldin 1.4821 and Calinski-Harabasz 970.75** - both acceptable at k=3.
4. **Business utility** - three segments can each own a campaign and a budget. Eight cannot.

### Final model

| Metric | Value | Direction |
| --- | ---: | --- |
| Silhouette | 0.2390 | higher is better |
| Davies-Bouldin | 1.4821 | lower is better |
| Calinski-Harabasz | 970.75 | higher is better |

Projected onto the first two principal components (46.1% of total variance combined), the
three clusters form a clear triangle. PC1 is effectively a buy-versus-borrow axis
(`PURCHASES_FREQUENCY` +0.52 at one end, `CASH_ADVANCE_FREQUENCY` -0.35 at the other) and
PC2 tracks balance carried (`BALANCE_FREQUENCY` +0.56, `BALANCE` +0.39), which is why the
dormant group sits low.

---

## Repository structure

```
P1G6_daffa_hutapea.ipynb        EDA, preprocessing, model training and evaluation
P1G6_daffa_hutapea_inf.ipynb    Inference on an unseen customer (separate notebook)
P1G6_daffa_hutapea.csv          Extracted dataset, 4,475 rows
preprocessing_pipeline.pkl      Fitted SimpleImputer + StandardScaler
pca.pkl                         Fitted PCA, 11 components
kmeans_model.pkl                Final K-Means, k=3
feature_columns.pkl             Locked column order for inference
dataset-description.png         Source data dictionary
ASSIGNMENT.md                   Original assignment brief
README.md
```

---

## Data

Pulled from Google BigQuery rather than loaded from a file, so the extract is reproducible:

```sql
SELECT
    *
FROM
    `ftds-hacktiv8-project.phase1_ftds_040_hck.credit-card-information`
WHERE
    MOD(CUST_ID, 2) = 0;
```

The `MOD` filter follows the assignment rule that even-numbered batches take the
even-`CUST_ID` half of the table, returning 4,475 of 8,950 rows. The result is saved to
`P1G6_daffa_hutapea.csv`, which is committed here so the notebooks run without BigQuery
credentials.

Each row is one cardholder's aggregated behavior over six months: balance, purchases split
into one-off and installment, cash advances, repayment behavior, credit limit and tenure.
There are no demographic fields. See `dataset-description.png` for the full data dictionary.

---

## Getting started

Requires Python 3.10 or newer.

```bash
git clone <your-repo-url>
cd p1-ftds-g6-s1-new-daphutapea
pip install pandas numpy scipy scikit-learn matplotlib seaborn yellowbrick jupyter
jupyter notebook P1G6_daffa_hutapea.ipynb
```

Developed against pandas 2.3, numpy 2.3, scipy 1.16, scikit-learn 1.7, matplotlib 3.10 and
seaborn 0.13. `yellowbrick` is used only for the silhouette overview grid; every other cell
runs without it.

The notebook is committed with outputs, so the results above are readable without
re-running anything.

---

## Scoring a new customer

Inference lives in `P1G6_daffa_hutapea_inf.ipynb` and reuses the four saved artifacts, so a
new row is transformed with exactly the statistics the model was trained on:

```python
import pandas as pd
import pickle

with open('preprocessing_pipeline.pkl', 'rb') as f:
    pipeline = pickle.load(f)
with open('pca.pkl', 'rb') as f:
    pca = pickle.load(f)
with open('kmeans_model.pkl', 'rb') as f:
    kmeans_model = pickle.load(f)
with open('feature_columns.pkl', 'rb') as f:
    feature_columns = pickle.load(f)

# feature_columns locks the column order, so one misplaced field cannot
# silently corrupt a prediction
new_df = pd.DataFrame([new_customer])[feature_columns]

predicted_cluster = int(kmeans_model.predict(pca.transform(
    pipeline.transform(new_df)))[0])
```

The worked example (`CUST_ID` 9999) lands in cluster 1, Active Transactors, at a centroid
distance of 8.07 against 9.96 and 10.49 for the other two.

That test record also carries `PURCHASES_FREQUENCY = 5`, which is outside the variable's
valid `[0, 1]` range since every frequency field is a share of months. It is scored as the
brief specifies, but flagged in the notebook: a production pipeline would clamp or reject
that row at validation.

---

## Limitations

- **Silhouette of 0.24 means the clusters genuinely overlap**, particularly between
  revolvers and mid-tier transactors. K-Means still assigns a hard label to every account,
  including ones sitting near a boundary.
- **K-Means assumes roughly spherical clusters of similar variance.** Financial behavior is
  not that tidy.
- **Mean imputation of 158 `MINIMUM_PAYMENTS` values introduces bias.** Accounts with a
  genuinely missing minimum payment may be edge cases rather than average ones.
- **85% of accounts sit at 12-month tenure**, which caps what any tenure-based analysis can
  resolve.
- **No demographic or temporal data**, so segments describe behavior only and cannot be
  cross-checked against customer lifecycle.

## Future work

1. **Test HDBSCAN.** Density-based clustering handles non-spherical shapes and can leave a
   genuine outlier unlabeled instead of forcing it into a segment.
2. **Validate against live campaign results.** If two segments respond identically to the
   same offer, they are one segment and should be merged.
3. **Add month-over-month deltas.** Tracking movement between segments rather than
   membership would turn this from a marketing tool into a risk signal: a transactor
   sliding into revolver behavior is an early warning.

---

## References

1. Rousseeuw, P. J. (1987). Silhouettes: a graphical aid to the interpretation and
   validation of cluster analysis. *Journal of Computational and Applied Mathematics*,
   20, 53-65.
2. Davies, D. L. and Bouldin, D. W. (1979). A Cluster Separation Measure. *IEEE Trans.
   Pattern Analysis and Machine Intelligence*, PAMI-1(2), 224-227.
3. Calinski, T. and Harabasz, J. (1974). A dendrite method for cluster analysis.
   *Communications in Statistics*, 3(1), 1-27.
4. scikit-learn, [`davies_bouldin_score`](https://scikit-learn.org/stable/modules/generated/sklearn.metrics.davies_bouldin_score.html)
5. scikit-learn, [`calinski_harabasz_score`](https://scikit-learn.org/stable/modules/generated/sklearn.metrics.calinski_harabasz_score.html)
6. Google Cloud, [BigQuery standard SQL query syntax](https://docs.cloud.google.com/bigquery/docs/reference/standard-sql/query-syntax)
7. Statistikian, [Penjelasan dan Teori Uji Kruskal Wallis H](https://www.statistikian.com/2014/07/kruskall-wallis-h.html)
8. Atlassian, [A Complete Guide to Violin Plot](https://www.atlassian.com/data/charts/violin-plot-complete-guide)
9. Plain English, [Data Visualization with Python](https://python.plainenglish.io/data-visualization-with-python-8ce66744ccbd)

---

## Author

**Daffa Narendra Hutapea**
Hacktiv8 Data Science Full-Time Program, Batch HCK-040

Built as Graded Challenge 6 (clustering). The bank in the brief is fictional; the data,
pipeline and results are real.
