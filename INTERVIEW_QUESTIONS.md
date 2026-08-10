# Interview Questions & Answers
# Customer Churn & Revenue Intelligence Analytics Project

---

## Section 1 — Python Questions

**Q1. How did you handle missing values in the TotalCharges column?**

> `TotalCharges` was stored as an object (string) because some rows contained only whitespace. I used `pd.to_numeric(df['TotalCharges'], errors='coerce')` to convert it, which turned blank strings into NaN. I then imputed the NaN values using `tenure × MonthlyCharges` — a business-logical estimate for customers with very low tenure. I chose this over mean imputation because the true value is mathematically deterministic for new customers.

---

**Q2. What is the difference between `df.fillna()` and imputing with a formula?**

> `fillna(mean)` replaces NaN with a single global average, which ignores the individual customer's context. A formula-based imputation like `tenure × MonthlyCharges` uses actual customer data, producing a more accurate estimate and maintaining internal consistency in the dataset.

---

**Q3. Why did you use `pd.cut()` for tenure binning instead of `pd.qcut()`?**

> `pd.cut()` uses fixed-width bins (0–12, 13–24, etc.) that correspond to business-meaningful lifecycle stages. `pd.qcut()` uses quantile-based bins which ensures equal-sized groups but doesn't align with natural business interpretations. For this project, business-interpretable segments are more important than equal bin sizes.

---

**Q4. Explain how `StandardScaler` works and when you applied it.**

> `StandardScaler` transforms features to have zero mean and unit standard deviation: `(x - mean) / std`. I applied it in the Logistic Regression pipeline because LR is sensitive to feature scale. I fit the scaler only on training data and applied it to both train and test to prevent data leakage.

---

**Q5. What is the purpose of `StratifiedKFold` in cross-validation?**

> `StratifiedKFold` ensures that each fold contains the same proportion of the target classes (churned vs retained) as the full dataset. This is critical for imbalanced datasets — without stratification, some folds might have very few or no churned customers, making cross-validation metrics unreliable.

---

**Q6. How did you handle class imbalance in the churn prediction models?**

> I used `class_weight='balanced'` in both Logistic Regression and Random Forest. This parameter weights each class inversely proportional to its frequency, effectively giving the minority class (churned customers, ~26%) more importance during training.

---

**Q7. What is a Pipeline in scikit-learn and why did you use it?**

> A `Pipeline` chains multiple processing steps (imputation → scaling → model) into a single object. Benefits: (1) prevents data leakage by ensuring fit-only-on-train is enforced automatically, (2) simplifies cross-validation, (3) makes the workflow reproducible and clean.

---

**Q8. Explain the difference between `LabelEncoder` and ordinal mapping.**

> `LabelEncoder` assigns arbitrary integer labels to categories, which can introduce unintended ordinal relationships. I used explicit ordinal mapping for ordered categoricals (e.g., `Contract`: Month-to-month=0, One year=1, Two year=2) to preserve the real ordering. For binary Yes/No columns, I mapped directly to 1/0.

---

**Q9. What does `n_init=10` mean in KMeans?**

> It means K-Means is run 10 times with different random centroid initializations. The result with the lowest inertia (within-cluster sum of squares) is kept. This reduces the risk of getting stuck in a local minimum from a poor initialization.

---

**Q10. How would you save and reload your trained scikit-learn model?**

> Using `joblib`: `joblib.dump(model, 'model.pkl')` to save, `joblib.load('model.pkl')` to reload. `joblib` is preferred over `pickle` for scikit-learn models because it handles NumPy arrays more efficiently.

---

## Section 2 — SQL Questions

**Q11. Explain the difference between WHERE and HAVING.**

> `WHERE` filters rows before grouping; it cannot reference aggregate functions. `HAVING` filters after grouping and can reference aggregates. Example: `SELECT contract, AVG(churn) FROM customers GROUP BY contract HAVING AVG(churn) > 0.3` — impossible with WHERE.

---

**Q12. What is a CTE and when is it better than a subquery?**

> A CTE (Common Table Expression) is a named temporary result set defined with `WITH`. CTEs improve readability (vs nested subqueries), can be referenced multiple times in the same query, and support recursive queries. Use CTEs when: the result is referenced more than once, or the query has multiple logical steps.

---

**Q13. Explain the difference between RANK(), DENSE_RANK(), and ROW_NUMBER().**

> All three assign a number to rows within a window. `ROW_NUMBER()` gives unique sequential numbers. `RANK()` gives the same number to ties but skips subsequent numbers (e.g., 1, 2, 2, 4). `DENSE_RANK()` gives the same number to ties without skipping (e.g., 1, 2, 2, 3). I used `RANK()` in the revenue analysis to rank customers within segments.

---

**Q14. What does PARTITION BY do in a window function?**

> `PARTITION BY` divides the dataset into groups (like GROUP BY) but does NOT collapse rows. The window function is computed separately within each partition. For example: `AVG(monthly_charges) OVER (PARTITION BY value_segment)` computes the segment-specific average next to each row without losing individual rows.

---

**Q15. How would you write a query to find the top 3 churners by monthly charges within each contract type?**

```sql
WITH ranked AS (
    SELECT customer_id, contract, monthly_charges, churn,
           RANK() OVER (PARTITION BY contract ORDER BY monthly_charges DESC) AS rnk
    FROM customer_churn
    WHERE churn = 1
)
SELECT * FROM ranked WHERE rnk <= 3;
```

---

**Q16. What is the difference between INNER JOIN and LEFT JOIN?**

> `INNER JOIN` returns only rows with matching values in both tables. `LEFT JOIN` returns all rows from the left table and matching rows from the right; unmatched right-side rows become NULL. I used the LEFT JOIN pattern when joining model_predictions to customer_churn to retain all customers even if predictions weren't generated.

---

**Q17. How did you implement a running total in SQL?**

> Using `SUM(total_charges) OVER (ORDER BY total_charges DESC ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)` — the `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` frame clause accumulates the sum from the first row to the current row.

---

**Q18. What is NTILE() and how did you use it?**

> `NTILE(n)` divides rows into n equal-sized groups. I used `NTILE(10)` (deciles) and `NTILE(5)` (quintiles) to analyse revenue concentration — identifying whether the top 20% of customers generate a disproportionate share of MRR (Pareto analysis).

---

**Q19. How would you calculate churn rate in SQL?**

```sql
SELECT ROUND(SUM(churn) * 100.0 / COUNT(*), 2) AS churn_rate_pct
FROM customer_churn;
```

---

**Q20. What indexes did you create and why?**

> I created B-tree indexes on `churn`, `contract`, `tenure`, `value_segment`, `business_segment`, `payment_method`, and `internet_service`. These columns are used in WHERE clauses and GROUP BY in the analysis queries. Indexes speed up these lookups, especially important for the analytical queries that filter on multiple dimensions simultaneously.

---

## Section 3 — Power BI Questions

**Q21. What is a DAX measure vs a calculated column?**

> A **calculated column** is computed row-by-row during data refresh and stored in the model — uses row context. A **DAX measure** is computed dynamically at query time based on filter context — does not store data. Measures are preferred for aggregations like `Churn Rate` because they respond to slicers and filters dynamically.

---

**Q22. How do you create a Churn Rate measure in Power BI?**

```dax
Churn Rate = 
DIVIDE(
    COUNTROWS(FILTER('powerbi_dataset', 'powerbi_dataset'[Churn] = 1)),
    COUNTROWS('powerbi_dataset'),
    0
) * 100
```

---

**Q23. What is the difference between CALCULATE and FILTER in DAX?**

> `FILTER` returns a filtered table. `CALCULATE` evaluates an expression in a modified filter context. `CALCULATE` is more powerful because it can add/modify/remove filters from the context: `CALCULATE([Total Customers], 'data'[Churn] = 1)` is equivalent to counting churned customers regardless of other slicers.

---

**Q24. What would you put on a Power BI Executive Overview page?**

> KPI cards: Total Customers, Churn Rate, Active MRR, Monthly Revenue at Risk, Retention Rate. Charts: Churn by Contract Type (bar), Revenue by Segment (treemap), Churn trend by Tenure Band (line or bar). This page should give a C-level executive the full picture in one glance.

---

## Section 4 — Statistics Questions

**Q25. What is the difference between mean and median? When would you use each?**

> Mean is the arithmetic average — sensitive to outliers. Median is the middle value — robust to outliers. For skewed distributions like `TotalCharges` (long-tail due to high-tenure customers), median is a better central tendency measure. I used both in the EDA to describe numerical distributions.

---

**Q26. What is the IQR method for outlier detection?**

> IQR = Q3 - Q1. Outliers are defined as values below `Q1 - 1.5×IQR` or above `Q3 + 1.5×IQR`. I applied this to `tenure`, `MonthlyCharges`, and `TotalCharges`. All flagged records were genuine customer observations (high-tenure long-term customers), not data errors, so they were retained.

---

**Q27. What is correlation and what does a correlation of 0.85 between tenure and TotalCharges mean?**

> Correlation measures the linear relationship between two variables, ranging from -1 to +1. A value of 0.85 between `tenure` and `TotalCharges` means they have a strong positive relationship: as tenure increases, total charges also increase. This is expected because customers who stay longer accumulate more charges over time.

---

**Q28. What is class imbalance and how does it affect model evaluation?**

> Class imbalance occurs when one class is far more common than the other. In churn data, ~74% of customers are retained. A model that always predicts "no churn" achieves 74% accuracy while being completely useless for business purposes. This is why we use Recall, Precision, F1-Score, and ROC-AUC instead of just accuracy.

---

**Q29. What is ROC-AUC and how do you interpret it?**

> ROC-AUC (Receiver Operating Characteristic - Area Under Curve) measures the model's ability to discriminate between classes across all classification thresholds. AUC = 0.5 means random performance; AUC = 1.0 means perfect discrimination. AUC = 0.83 (example) means the model has an 83% chance of correctly ranking a churner above a non-churner.

---

## Section 5 — Machine Learning Questions

**Q30. Why did you choose Logistic Regression and Random Forest for this problem?**

> Logistic Regression is interpretable, fast, provides probability outputs, and serves as a strong baseline. Random Forest handles non-linear relationships, is robust to outliers, requires minimal hyperparameter tuning, and provides feature importance. Together they give a comprehensive view of model performance from simple to complex.

---

**Q31. What is the trade-off between Precision and Recall in churn prediction?**

> Precision = True Positives / (True Positives + False Positives) — of those flagged as churners, how many actually churned? Recall = True Positives / (True Positives + False Negatives) — of all actual churners, how many were caught? In churn prediction, higher Recall is preferred because missing a churner (false negative) costs more than incorrectly flagging a retained customer (false positive).

---

**Q32. What is feature importance in Random Forest?**

> Random Forest measures feature importance by calculating how much each feature reduces impurity (Gini impurity) across all trees. Features that split high-impurity nodes near the root are most important. In this project, `tenure`, `Contract_Enc`, and `MonthlyCharges` typically rank highest.

---

**Q33. What is overfitting and how did you check for it?**

> Overfitting occurs when a model learns training data patterns too closely and performs poorly on unseen data. I checked using 5-fold stratified cross-validation: if cross-validation ROC-AUC was close to test set ROC-AUC, the model generalises well. Large differences (e.g., train AUC 0.95, test AUC 0.70) would indicate overfitting.

---

## Section 6 — Business Analytics Questions

**Q34. How do you define Customer Lifetime Value in this project?**

> Two CLV versions: (1) **Historical CLV** = `TotalCharges` — actual money received from the customer to date. (2) **Projected CLV** = `AvgMonthlyRevenue × 24 months` — a forward-looking 2-year estimate using the customer's average monthly spend as a proxy for future value.

---

**Q35. What is MRR and why is it important?**

> MRR (Monthly Recurring Revenue) = sum of all active customers' monthly charges. It is the fundamental health metric for subscription businesses. MRR tells you the predictable revenue baseline. "Monthly Revenue at Risk" = MRR from active high-risk customers, telling you what you could lose next month if they churn.

---

**Q36. How would you prioritise which customers to target for a retention campaign?**

> Using the 2×2 matrix: **Value × Risk**. Priority order: (1) High Value + High Risk — biggest ROI, prevent immediately. (2) High Value + Low Risk — protect proactively with loyalty programs. (3) Low Value + High Risk — low-cost digital outreach. (4) Low Value + Low Risk — monitor only.

---

## Section 7 — Project-Specific Questions

**Q37. If your churn rate calculation is 26.5%, what does that mean for the business?**

> It means roughly 1 in 4 customers left during the observation period. This is above the typical telecom industry average of 15–25% annually. It signals an urgent need for retention investment, especially in the month-to-month and fiber optic segments where the rate is significantly higher.

---

**Q38. How did you decide on 4 clusters for K-Means segmentation?**

> I used two methods: the Elbow Method (inertia starts decreasing slowly at k=4) and the Silhouette Score (measures how well-separated clusters are — higher is better). Both pointed to k=4 as the optimal balance between cluster granularity and separability. The 4 clusters also mapped naturally to the 2×2 Value × Risk business matrix.

---

**Q39. What would you recommend to the business as the single highest-impact action?**

> Launch a **contract upgrade campaign** targeting month-to-month customers with 12+ months tenure. These customers have already proven loyalty but face the highest churn risk. Converting them to annual contracts reduces their month-to-month flexibility, and the data shows two-year contract customers churn at a fraction of the rate. Even a 15% conversion rate would protect significant MRR.

---

**Q40. How would you measure the success of the retention strategies you recommended?**

> For each strategy: (1) Define a control group (no intervention) vs treatment group (intervention). (2) Track churn rate over 3, 6, and 12 months post-campaign. (3) Calculate incremental revenue saved = (control churn rate - treatment churn rate) × group size × ARPU. (4) Compare to campaign cost to compute ROI. This is an A/B test framework for retention programs.

---

*Prepared for Data Analyst fresher interviews. All answers are based on actual project implementation.*
