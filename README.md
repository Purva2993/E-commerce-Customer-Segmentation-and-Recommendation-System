# E-commerce Customer Segmentation & Product Recommendation

An end-to-end data science project that analyzes historical online retail transactions, segments customers by purchasing behavior, and compares three product recommendation strategies. The main question is whether recommendations based on **products purchased together** can outperform simpler popularity-based recommendations on a time-based holdout dataset.

> **Main finding:** In the offline evaluation, item-to-item co-purchase recommendations outperformed global and customer-segment popularity baselines on Precision@10, Recall@10, and Hit Rate@10. This indicates better ranking performance on this test set; it does not demonstrate increased sales or production impact.

## Table of Contents

- [Project Goals](#project-goals)
- [Dataset](#dataset)
- [Workflow](#workflow)
- [Customer Segmentation](#customer-segmentation)
- [Recommendation Methods](#recommendation-methods)
- [Evaluation Results](#evaluation-results)
- [Robustness and Stability Checks](#robustness-and-stability-checks)
- [Repository Structure](#repository-structure)
- [How to Run](#how-to-run)
- [Limitations](#limitations)
- [Potential Next Steps](#potential-next-steps)
- [Acknowledgements](#acknowledgements)

## Project Goals

1. Clean transaction-level retail data and construct meaningful customer-level purchasing features.
2. Use **Principal Component Analysis (PCA)** and **K-Means clustering** to identify groups with different purchasing activity.
3. Build and compare three product recommendation methods:
   - Global popularity
   - Segment popularity
   - Item-to-item co-purchase associations
4. Evaluate the recommendations using a chronological train/test split, ranking metrics, paired-bootstrap confidence intervals, and a within-test-period stability check.

## Dataset

This project uses the **Online Retail** transaction dataset from the [UCI Machine Learning Repository](https://archive.ics.uci.edu/dataset/352/online+retail). It contains transactions from a UK-based online retailer between December 2010 and December 2011. The original dataset has **541,909 rows and 8 columns**.

The fields used in the analysis include:

| Field | Description |
|---|---|
| `InvoiceNo` | Invoice identifier; values beginning with `C` indicate cancellations. |
| `StockCode` | Product identifier. |
| `Description` | Product description. |
| `Quantity` | Number of units on a transaction line; negative values can represent returns or cancellations. |
| `InvoiceDate` | Transaction date and time. |
| `UnitPrice` | Unit price in pounds sterling. |
| `CustomerID` | Customer identifier; some transaction rows do not have one. |
| `Country` | Customer's country. |

The raw dataset is not included in this repository. Download it from UCI and prepare the CSV file expected by the notebook as described in [How to Run](#how-to-run). Please follow the dataset's attribution requirements when reusing the data.

## Workflow

The notebook follows these main steps:

1. **Data audit and cleaning** — inspect missing values, data types, exact duplicates, cancellation records, and special transaction codes.
2. **Customer-level feature engineering** — summarize purchase recency, purchase frequency, merchandise spend, units, and product variety.
3. **Feature transformation** — apply `log1p` to reduce right skew and standardize the features before PCA and clustering.
4. **Customer segmentation** — reduce the feature space with PCA and use K-Means to form three interpretable activity groups.
5. **Recommendation development** — create global, segment-based, and item-to-item recommenders using the training period only.
6. **Offline evaluation** — compare the top 10 recommendations against qualifying purchases in a later test period.
7. **Robustness checks** — compare recommendation performance in two portions of the holdout and examine selected preprocessing sensitivities.

### Data preparation notes

- Removed **5,268 exact duplicate rows**, leaving **536,641 rows** after duplicate removal.
- Preserved the broad cleaned transaction table, while using transactions with known customer IDs for customer-level modeling.
- Investigated cancellations instead of treating all negative quantities as ordinary purchases.
- Excluded special transaction codes (such as fees, postage, and bank charges) from merchandise recommendation features.
- Used positive, eligible merchandise purchase events for recommendation baskets and rankings. Negative merchandise lines were retained in the net-spend calculation so that returns could affect net spend.
- Used a documented invoice-matching heuristic to identify likely full purchase reversals within 24 hours. This is an approximation, not proof that each matched transaction was reversed.

## Customer Segmentation

### Features

The main customer segmentation model uses five features:

- **Recency:** days since the most recent eligible purchase.
- **Frequency:** number of distinct eligible purchase invoices.
- **Gross merchandise spend:** total spend over eligible positive purchase events.
- **Total units:** total units purchased.
- **Unique products:** number of distinct products purchased.

The five features were log-transformed with `log1p`, standardized with `StandardScaler`, and reduced to two principal components. In the full customer-feature analysis, these two components retained **85.91% of the variance**.

Three K-Means clusters were selected for interpretability. A two-cluster solution scored better on silhouette separation, so three clusters represent a practical trade-off for producing low-, moderate-, and high-activity groups—not a claim that three clusters are uniquely optimal.

### Customer profiles

The following profiles describe the full customer-feature segmentation. Monetary values are in pounds sterling and are medians, not averages.

| Segment | Customers | Share | Median recency | Median purchase frequency | Median gross spend | Median units | Median distinct products | Median average order value |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| Low activity | 1,666 | 38.45% | 150 days | 1 | £240.62 | 129 | 14 | £192.58 |
| High activity | 908 | 20.96% | 11 days | 8 | £3,279.12 | 1,959.5 | 121 | £386.93 |
| Moderate activity | 1,759 | 40.60% | 44 days | 3 | £873.74 | 505 | 47 | £317.18 |

**Interpretation:** Low-activity customers generally bought less recently and less frequently. High-activity customers had more recent purchases, higher purchase frequency, greater spend, and a wider range of products. Moderate-activity customers fell between the other two groups on these measures. These are descriptive purchasing-behavior segments, not definitive labels for customer preferences or value.

## Recommendation Methods

All three methods use eligible transactions from the training period and exclude products the customer had already purchased in that period.

### 1. Global popularity

Rank products by the number of distinct customers who purchased them, using the number of distinct invoices as a secondary ranking criterion. Recommend the highest-ranked products the customer has not already purchased.

**Question:** What products are popular across the whole store?

### 2. Segment popularity

Create the same popularity ranking separately within each training-period customer cluster. Use the ranking for the customer's assigned cluster and skip products already purchased by that customer.

**Question:** What products are popular among customers in the same purchasing-activity group?

### 3. Item-to-item co-purchase associations

Treat each training-period invoice as a shopping basket. Count product pairs that appear together in at least **three invoices** and calculate directional association confidence:

`Confidence(A → B) = invoices containing both A and B / invoices containing A`

For a customer, gather candidate products associated with the products in their purchase history, remove previously purchased items, combine association-confidence scores across their history, and rank the candidates using the combined score, with pair support as a tie-breaker.

**Question:** What products tend to be purchased alongside items this customer has already bought?

Co-purchase patterns show association, not causation. They do not prove that purchasing one product causes someone to purchase another.

## Evaluation Results

### Time-based holdout

The recommendation experiment used a chronological split with a cutoff of **October 27, 2011, at 14:52:48**. The recommendation rankings and training-only preprocessing were built using earlier data, then evaluated against later purchases.

| Partition | Invoices | Transaction lines | Customers represented |
|---|---:|---:|---:|
| Training | 14,682 | 304,144 | 3,930 |
| Test | 3,669 | 86,703 | 1,998 |

The overall recommendation comparison included **1,505 eligible customers**. Eligibility required a customer to have training-period purchase history and at least one distinct product bought in the test period that was not present in their training-period purchase history. Metrics are averaged at the customer level.

### Overall ranking metrics at K = 10

| Recommendation method | Precision@10 | Recall@10 | Hit Rate@10 |
|---|---:|---:|---:|
| Global popularity | 3.45% | 1.79% | 25.25% |
| Segment popularity | 3.77% | 2.08% | 27.18% |
| **Item-to-item associations** | **4.85%** | **3.47%** | **31.63%** |

**Metric definitions**

- **Precision@10:** Of the 10 recommended products, the average proportion that matched a qualifying product bought in the test period.
- **Recall@10:** The average proportion of the customer's qualifying test-period products that appeared in the top 10 recommendations.
- **Hit Rate@10:** The proportion of eligible customers who received at least one relevant recommendation.

Item-to-item associations achieved the highest score for all three metrics in this holdout.

### Paired-bootstrap confidence intervals

We compared item-to-item recommendations with each baseline using **3,000 paired bootstrap resamples at the customer level**. Improvements are expressed in percentage points (pp).

| Metric | Baseline | Improvement | 95% bootstrap confidence interval |
|---|---|---:|---:|
| Precision@10 | Global popularity | +1.402 pp | +1.017 to +1.767 pp |
| Precision@10 | Segment popularity | +1.083 pp | +0.691 to +1.455 pp |
| Recall@10 | Global popularity | +1.678 pp | +1.185 to +2.209 pp |
| Recall@10 | Segment popularity | +1.390 pp | +0.927 to +1.891 pp |
| Hit Rate@10 | Global popularity | +6.379 pp | +4.252 to +8.439 pp |
| Hit Rate@10 | Segment popularity | +4.452 pp | +2.392 to +6.645 pp |

All six confidence intervals were above zero. This supports a positive item-to-item improvement within this evaluation sample. The intervals do not estimate performance across entirely new future periods and do not establish real-world sales impact.

## Robustness and Stability Checks

### Earlier and later portions of the same test period

The 3,669 test invoices were split by invoice time into two approximately equal windows around **November 18, 2011, at 13:16**. This checks whether the result is concentrated in only one part of the existing holdout; it is not an independent second test set.

| Test window | Eligible customers | Method | Precision@10 | Recall@10 | Hit Rate@10 |
|---|---:|---|---:|---:|---:|
| Earlier | 1,014 | Global popularity | 3.16% | 1.93% | 23.57% |
| Earlier | 1,014 | Segment popularity | 3.54% | 2.24% | 26.23% |
| Earlier | 1,014 | **Item-to-item associations** | **4.42%** | **3.64%** | **29.39%** |
| Later | 950 | Global popularity | 2.56% | 1.57% | 20.00% |
| Later | 950 | Segment popularity | 2.59% | 1.60% | 19.79% |
| Later | 950 | **Item-to-item associations** | **3.52%** | **3.04%** | **24.63%** |

Item-to-item associations ranked first on all three metrics in both windows. Scores were lower for all methods in the later window. The current analysis does not establish the reason for the decrease. Customer counts across windows are not additive because a customer may appear in both. For this stability check, the customer's training-period history is used as the known history, so an item purchased in the earlier test window may still count as relevant if it is purchased again in the later window.

### Segmentation and preprocessing sensitivity

- PCA-based cluster assignments were highly similar to those from the standardized five-feature representation (adjusted Rand index: **0.984 for K=2** and **0.908 for K=3** in the comparisons performed).
- Capping features at the 99th percentile before transformation and clustering produced an adjusted Rand index of **0.9232** against the baseline three-cluster assignments. After aligning cluster labels, **97.3%** of customers stayed in the corresponding segment.
- Removing the 24-hour full-invoice reversal heuristic resulted in an adjusted Rand index of **0.9906** versus the baseline cluster assignments. The silhouette score was **0.3812** for the baseline and **0.3814** without the rule.

These checks suggest the broad clustering structure was similar under the alternatives tested. The cancellation rule remains a heuristic and does not guarantee that every reversal was correctly identified.

## Repository Structure

A simple repository layout is:

```text
.
├── README.md
├── E_commerce_Customer_Segmentation.ipynb
├── data.csv                         # downloaded/prepared locally; do not commit if you prefer to keep raw data out of Git
└── outputs/                         # optional: exported result files
    ├── ecommerce_customer_segments_final.csv
    ├── ecommerce_customer_segment_profiles.csv
    └── ecommerce_customer_segmentation_evaluation.csv
```

The notebook and the exported CSV files are separate artifacts. Add the output files to your repository only if they are present and you intend to share them. The raw dataset is not required to be committed to GitHub.

## How to Run

The notebook currently loads a CSV with this line:

```python
customer_df = pd.read_csv("/content/data.csv", encoding="latin1")
```

`/content/data.csv` is a Google Colab-style path. To run the notebook in Google Colab, upload the prepared CSV to `/content/data.csv`. To run locally, put `data.csv` in the notebook's working directory and change the path to `"data.csv"`, or set it to the actual file location.

The official UCI download is provided as an Excel workbook. If you use that workbook directly rather than a CSV, update the loading cell to use `pd.read_excel(...)` and install `openpyxl` if needed.

### Install dependencies

Use Python 3.11 or another compatible Python 3.x version, then install the notebook's main libraries:

```bash
python -m venv .venv
source .venv/bin/activate        # macOS/Linux
# .venv\Scripts\activate       # Windows

python -m pip install --upgrade pip
python -m pip install pandas numpy matplotlib seaborn scipy scikit-learn plotly jupyter
```

Then launch Jupyter and open the notebook:

```bash
jupyter notebook
```

Run the notebook cells in order. Runtime and results may vary with changes to the input data, preprocessing decisions, package versions, and random seeds.

## Limitations

- **Historical dataset:** The transactions are from 2010–2011 and may not reflect current retail behavior or catalog availability.
- **Offline evaluation only:** There was no live experiment, and the project did not measure clicks, conversions, incremental revenue, customer satisfaction, production latency, or operating cost.
- **One main temporal holdout:** Bootstrap resampling estimates uncertainty across customers in this test sample. The two-window check reuses the same holdout and is not independent validation on a new period.
- **Eligible-customer subset:** The overall metrics cover 1,505 eligible returning customers, not every customer or cold-start user.
- **Relevance definition:** A product not purchased in the test period is considered not relevant for this evaluation, even though the customer may purchase it outside the observed period.
- **Cancellation handling:** The 24-hour full-invoice reversal matching rule is a heuristic; it may misclassify some transactions.
- **Association is not causation:** Products appearing together in invoices are not proof of a causal relationship.
- **Three-cluster choice:** Three clusters were retained for interpretation, even though K=2 had better silhouette separation in the principal comparison.

## Potential Next Steps

- Evaluate on one or more untouched future periods using rolling chronological validation.
- Revisit the relevance definition and account for repeat purchases across test windows.
- Test different minimum co-purchase support levels and recommendation-ranking formulas.
- Assess recommendation coverage, diversity, and cold-start behavior for customers with little history.
- If the model is ever deployed, monitor data drift and operational performance and use a well-designed online experiment to measure business outcomes.

## Acknowledgements

- **Dataset:** Chen, D. (2015). *Online Retail*. UCI Machine Learning Repository. [https://doi.org/10.24432/C5BW33](https://doi.org/10.24432/C5BW33). Dataset page: [UCI Online Retail](https://archive.ics.uci.edu/dataset/352/online+retail).
- **Tools:** Python, pandas, NumPy, scikit-learn, SciPy, Matplotlib, Seaborn, Plotly, and Jupyter Notebook.
