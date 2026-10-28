E-commerce Customer Segmentation and Product Recommendation

1. Project objective
This project uses historical online retail transactions to identify groups of customers with different purchasing behaviors and to test product recommendation strategies. The work has two connected parts:
1. Customer segmentation: group customers based on purchasing activity.
2. Product recommendation: compare different ways to recommend products a customer has not previously purchased.
The goal is to evaluate whether product-to-product purchasing relationships provide more useful recommendations than simple popularity-based approaches. This is an offline analysis; it does not measure live sales uplift or prove that recommendations would change customer behavior.
2. Data preparation and transaction handling
The original dataset contained 541,909 transaction rows and 8 columns. Exact duplicate rows were removed, leaving 536,641 rows. Customer-level analyses were restricted to transactions with a known CustomerID; transaction rows without customer IDs were preserved in the broader cleaned data rather than assigned to invented customers.

Before building customer features and recommenders, the workflow also:
- Converted transaction dates to a consistent datetime type and normalized customer identifiers.
- Distinguished merchandise purchases from special transaction codes such as postage, fees, discounts, and bank charges.
- Investigated cancellation records rather than treating every negative quantity as an ordinary purchase.
- Used a conservative, invoice-level matching heuristic to identify likely full reversals occurring within 24 hours. This is a heuristic, not proof that every matched invoice was a true reversal.
- Built the recommendation training data from eligible positive merchandise purchase events and excluded products a customer had already purchased from that customer's recommendations.
For net merchandise spend, negative merchandise return/cancellation lines were retained so that returns could affect net spend. For the positive purchase-event dataset used to construct the recommendation rankings and baskets, cancellation invoices and special transaction codes were excluded according to the documented rules.
3. Customer segmentation: PCA and K-Means
  
Features used
The initial segmentation model used five customer-level features:
- Recency_Days: days since the customer's most recent eligible purchase.
- Frequency: number of distinct eligible purchase invoices.
- Gross_Merchandise_Spend: total spend across eligible positive purchase events.
- Total_Units: total units purchased across those events.
- UniqueProducts: number of distinct products purchased.
The features were log-transformed with log1p to reduce skew and then standardized. PCA was used to reduce the five standardized features to two principal components. Those two components explained 85.91% of the variance in the full customer-feature analysis. K-Means with three clusters was selected to create interpretable low-, moderate-, and high-activity groups.

The two-cluster solution had a higher silhouette score, so choosing three clusters was a practical interpretability trade-off rather than a claim that three was the only statistically possible choice. The clusters describe purchasing activity; they should not be interpreted as proof that customers in a cluster have identical preferences.
Customer segment profiles
The profiles below are based on the full customer-feature analysis and use the original, uncapped feature values for interpretation.
Segment	Customers	Share	Median recency	Median frequency	Median gross spend	Median units	Median distinct products	Median average order value
Low activity	1,666	38.45%	150 days	1	240.62	129	14	192.58
High activity	908	20.96%	11 days	8	3,279.12	1,959.5	121	386.93
Moderate activity	1,759	40.60%	44 days	3	873.74	505	47	317.18


Interpretation: The low-activity group generally bought less recently and less frequently, while the high-activity group bought more recently, more frequently, and across more products. The moderate group fell between them on these measures. Segment labels are descriptive labels assigned after inspecting the profiles; numeric cluster IDs themselves have no inherent order.

Segmentation robustness checks
- Cluster assignments from PCA-based clustering were highly similar to assignments using the standardized five-feature representation (adjusted Rand index: 0.984 for K=2 and 0.908 for K=3 in the comparison performed).
- A 99th-percentile feature-capping sensitivity run retained an adjusted Rand index of 0.9232 against the baseline K=3 assignments. After matching cluster labels, 97.3% of customers remained in their corresponding segment. This suggests some sensitivity to extreme values but not a wholesale change in the segmentation.
  
4. Recommendation methods
All three methods were evaluated using training-period transactions. The training/test split was chronological so that recommendations were built from earlier purchases and assessed against later purchases.
A. Global popularity baseline
For each product (StockCode), the training data was used to count the number of distinct customers who purchased it and the number of distinct invoices containing it. Products were ranked primarily by distinct-customer count and secondarily by invoice count. For a given customer, products already purchased in the training period were skipped, and the next highest-ranked unseen products were returned.

Question answered: Which products were popular across the whole store?
B. Segment popularity baseline
The same popularity ranking was created separately within each customer cluster from the training-period cluster assignment. Recommendations for a customer came from the ranking associated with that customer's training cluster, again excluding products already purchased by that customer.

Question answered: Which products were popular among customers in this purchasing-behavior group?
C. Item-to-item co-purchase associations
Training-period invoices were treated as shopping baskets. Product pairs appearing together in at least three invoices were retained as associations. A directional confidence score was calculated as:

Confidence(A → B) = invoices containing A and B / invoices containing A

For a customer, the recommender gathered candidate products associated with the customer's previously purchased products, excluded products already purchased, combined association-confidence scores across the customer's history, and ranked candidates using the combined score with pair support as a tie-breaker.

Question answered: Which products tend to appear in the same shopping baskets as products this customer has purchased?

These co-purchase relationships indicate observed patterns; they do not establish that one product caused another product to be purchased.

5. Time-based holdout evaluation
The recommendation evaluation used a time-based split with a cutoff of October 27, 2011, at 14:52:48.
Partition	Invoices	Transaction lines	Customers represented
Training period	14,682	304,144	3,930
Test period	3,669	86,703	1,998

The recommendation model was rebuilt using training-period data only. The training-only scaler and PCA were fitted on the training customers; the two PCA components retained 84.86% of variance in that training-only feature set. The same time-based training setup was used to construct the global, segment, and item-to-item recommendation rankings.

The overall metric comparison covered 1,505 eligible customers. Eligibility required training-period purchase history and at least one distinct product purchased in the test period that was not present in the customer's training-period purchase history. The metrics were averaged at the customer level.
Overall recommendation results at K=10
Method	Precision@10	Recall@10	Hit Rate@10
Global popularity	3.45%	1.79%	25.25%
Segment popularity	3.77%	2.08%	27.18%
Item-to-item association	4.85%	3.47%	31.63%

- Precision@10 is the average proportion of the 10 recommended items that matched relevant products bought in the test period.
- Recall@10 is the average proportion of the customer's relevant test-period products that appeared in the top 10 recommendations.
- Hit Rate@10 is the proportion of eligible customers who had at least one recommended product match a relevant test-period purchase.
In this evaluation, item-to-item associations achieved the best result on all three metrics. These are offline ranking metrics, not conversion rate or revenue measurements.

6. Paired-bootstrap uncertainty estimates
We compared item-to-item results with each baseline using 3,000 paired customer-level bootstrap resamples. Improvements are expressed in percentage points (pp), and the confidence intervals describe uncertainty in the mean customer-level difference within this holdout evaluation.
Metric	Baseline	Improvement from item-to-item	95% bootstrap CI
Precision@10	Global popularity	+1.402 pp	+1.017 to +1.767 pp
Precision@10	Segment popularity	+1.083 pp	+0.691 to +1.455 pp
Recall@10	Global popularity	+1.678 pp	+1.185 to +2.209 pp
Recall@10	Segment popularity	+1.390 pp	+0.927 to +1.891 pp
Hit Rate@10	Global popularity	+6.379 pp	+4.252 to +8.439 pp
Hit Rate@10	Segment popularity	+4.452 pp	+2.392 to +6.645 pp

All six intervals were above zero. This supports the conclusion that item-to-item associations performed better than both baselines across these metrics for the evaluated customers. The intervals do not quantify uncertainty across new future time periods and should not be interpreted as proof of live business impact.

7. Stability check within the held-out test period
To check whether the item-to-item advantage was concentrated in only one portion of the holdout, the 3,669 test invoices were divided by their invoice dates into two approximately equal windows around November 18, 2011, at 13:16. This was a further analysis of the same test period, not a second independent train/test experiment.
Test window	Eligible customers	Method	Precision@10	Recall@10	Hit Rate@10
Earlier	1,014	Global popularity	3.16%	1.93%	23.57%
Earlier	1,014	Segment popularity	3.54%	2.24%	26.23%
Earlier	1,014	Item-to-item association	4.42%	3.64%	29.39%
Later	950	Global popularity	2.56%	1.57%	20.00%
Later	950	Segment popularity	2.59%	1.60%	19.79%
Later	950	Item-to-item association	3.52%	3.04%	24.63%


Item-to-item associations had the highest values for all three metrics in both windows. Scores for all methods were lower in the later window. That difference could reflect changes in the products or customers represented in each period; the current analysis does not identify its cause.

The customer counts are not additive: some customers can appear in both windows. For this check, relevance was defined relative to training-period history, so a product purchased in the earlier test window could also count as relevant in the later window if it was purchased again. This is a within-holdout stability check, not independent validation.

8. Sensitivity to cancellation handling
The quick full-invoice reversal heuristic was also tested by rerunning segmentation without the 24-hour rule. The resulting customer segment assignments had an adjusted Rand index of 0.9906 compared with the baseline assignments; the silhouette score was 0.3812 for baseline and 0.3814 without the rule. In the comparison, 48 customers and 439 transaction lines were affected, and one additional customer had eligible purchase events in the alternative setup.

This sensitivity result suggests the overall cluster structure was very similar under that alternative rule. It does not prove that every cancellation match was correct; the rule remains a documented heuristic.

9. Limitations
  1. Historical and offline data: The transactions cover 2010–2011. Customer behavior and product availability may differ today.
  2. One main temporal holdout: The bootstrap resamples customers within the same test period. The two-window check reuses that holdout and the same training-built recommendation methods; neither is independent validation on a new period.
  3. Eligible-customer subset: The main evaluation includes customers with training history and at least one qualifying new-to-customer product in the test period. Results do not represent every customer, including cold-start users.
  4. Test-period target definition: A product not purchased during the test period is counted as not relevant for this offline test, even though the customer might buy it in another period. In the two-window check, earlier test-window purchases are not added to the history used to exclude already-seen products in the later window.
  5. Heuristic transaction cleaning: The 24-hour full-invoice reversal rule is an approximation; matching transaction signatures does not prove that a purchase was truly reversed.
  6. Co-purchase is not causality: Item-to-item associations identify products purchased together, not why they were purchased together.
  7. No live outcome evidence: We did not run an online A/B test or measure clicks, conversions, incremental revenue, customer satisfaction, operational latency, or production costs.
   
10. Conclusion
The project combined PCA and K-Means customer segmentation with three recommendation strategies: global popularity, segment popularity, and item-to-item co-purchase associations. On the evaluated time-based holdout of 1,505 eligible customers, item-to-item associations achieved the highest Precision@10 (4.85%), Recall@10 (3.47%), and Hit Rate@10 (31.63%). Paired-bootstrap 95% confidence intervals for improvements over both popularity baselines were above zero, and the method also ranked first on all three metrics in both halves of the existing test period.

Project decision: Keep item-to-item associations as the leading offline recommendation approach for further validation, with global and segment popularity retained as baselines. The evidence supports better offline recommendation ranking on this dataset; it does not establish increased sales or readiness for production deployment.

11. Recommended next steps before real-world use
- Validate on one or more later, untouched time periods, ideally using rolling chronological evaluation.
- Review the relevance definition and cancellation-handling assumptions, including how repeat purchases across test windows are treated.
- Compare association support thresholds and test whether scoring should account for product popularity, confidence reliability, or recommendation diversity.
- Define cold-start behavior for customers with little or no history and products with few transactions.
- If a business deployment becomes possible, monitor latency, coverage, diversity, and catalog changes, then use an appropriately designed online experiment to assess business outcomes.
