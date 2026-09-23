# Team Name: APT

## Contributors

- Aesha -- `Aesha_APT_linear_regression.ipynb`
- Temesgen Mesfin Tewolde -- `tem_apt_logistic_regression.ipynb`
- Pranshul -- `Pranshul_APT_gam.ipynb`

## Dataset

The IBM Telco Customer Churn dataset contains 7,043 rows, each representing one customer, with 20 predictor columns covering demographics (gender, senior citizen status, dependents), account details (tenure, contract type, payment method, monthly and total charges), and subscribed services (phone, internet, streaming). The target column, Churn, is binary: Yes if the customer left within the last month, No otherwise. Roughly 26.5% of customers churned. The prediction task is to identify which customers are at risk of leaving so the company can act before they do.

## Assumption Checks

| Model | Key Assumptions Checked | Evidence | Concern |
|---|---|---|---|
| Linear regression | Linearity of predictors vs response; no multicollinearity | Ramsey RESET test (F=206, p<0.0001) and residuals-vs-fitted plot; correlation heatmap flagged TotalCharges, which was dropped | Linearity assumption is violated: RESET test strongly rejects the linear functional form. Predictions also fall outside [0,1], which is invalid for a binary outcome. |
| Logistic regression | Binary outcome; independence of observations; no perfect separation; limited multicollinearity; sufficient sample size | Outcome confirmed binary; Durbin-Watson score ~2.01 with no pattern in residuals vs order; max feature-churn correlation below 0.95; VIF screen found MonthlyCharges VIF=229, TotalCharges VIF=47, tenure VIF=43, PhoneService VIF=38 (TotalCharges dropped); 7,043 rows well exceeds 10-20 events per predictor | Severe multicollinearity remains after dropping TotalCharges. MonthlyCharges, tenure, and PhoneService still show high VIF, making individual coefficient estimates unstable. The linear log-odds assumption is also not formally verified. |
| GAM | Correct response family and class balance; coverage across continuous predictor ranges; multicollinearity between continuous predictors; linearity vs curvature in log-odds for each continuous predictor | Class balance: 26.5% churn (moderate imbalance); histograms showed full, dense coverage for tenure (1-72 months) and MonthlyCharges ($18-$119); TotalCharges correlated r=0.83 with tenure and r=0.65 with MonthlyCharges and was dropped; empirical logit plots showed a clear non-linear curve for tenure and non-monotonic pattern for MonthlyCharges | Additivity is assumed but not verifiable from EDA alone. The pygam library warns that p-values are unreliable when smoothing parameters are estimated from the data rather than fixed in advance. |

## Model Comparison

| Model | Performance Evidence | Interpretability Strength | Interpretability Weakness |
|---|---|---|---|
| Linear regression | RMSE 0.38 on test set; thresholded accuracy 0.80 at a 0.5 cutoff (not a valid probability model) | Coefficients are easy to read: a one-standard-deviation increase in a feature shifts the predicted churn score by the coefficient amount | Designed for continuous outcomes, not binary ones. Predictions fall outside [0,1] and have no probability interpretation. The RESET test showed functional form misspecification, so coefficients likely absorb non-linear signal incorrectly. |
| Logistic regression | Accuracy ~0.73; ROC-AUC ~0.79 on test set | Coefficients are interpretable as log-odds shifts; exponentiated values give odds ratios that are familiar to business audiences | Assumes each predictor relates linearly to the log-odds of churn. This forces a constant rate of change across the whole range of tenure, missing the steep early drop in churn risk that the GAM reveals. Multicollinearity makes individual coefficients unstable. |
| GAM | ROC-AUC ~0.844 on test set, compared to ~0.838 for plain logistic regression on the same split; recall on the churner class ~0.53 at the default 0.5 threshold | Smooth plots for tenure and MonthlyCharges show exactly where in the predictor range churn risk rises or falls, without assuming a straight line. The tenure curve clearly shows risk concentrates in the first 18 months and then flattens, a finding logistic regression would hide behind a single slope. | Additivity hides interactions (for example, the tenure effect may differ by contract type). The MonthlyCharges smooth term was not statistically significant (p=0.48), meaning the apparent curve may be noise. Dropping TotalCharges removes any direct commentary on total customer spend. |

## Recommendation

Recommended model: GAM

Why this model: The GAM achieves the best ROC-AUC of the three models and, more importantly, it correctly represents where churn risk actually lives in the data. The tenure smooth term (EDoF 8.4, p<0.001) shows that churn risk is steep and non-linear in the first 18 months, then flattens dramatically. A linear model forces a constant slope across all of tenure and therefore understates urgency early and overstates it later. The GAM makes this non-linearity visible in a plot any analyst can read, without giving up the interpretability of the additive structure.

What the company can responsibly conclude: Customers on month-to-month contracts, using fiber optic internet, paying by electronic check, and in their first 18 months of tenure are the highest-risk group for churn. These associations are stable across all three models, so the signal is credible. The first year and a half is the window where retention investment will yield the most return.

What the company should not conclude yet: That any of these associations are causal. Artificially extending a customer's nominal tenure, for example by auto-renewing a contract, will not reduce their churn probability by the amount the tenure curve suggests. The curve describes who survived, not what would happen if you intervened. The MonthlyCharges smooth effect was not statistically significant and should not be used to justify pricing decisions. Results also do not account for clustered behavior such as households, regional outages, or promotional cohorts.

One next analysis we would run: Fit a GAM or logistic regression with an explicit interaction term between tenure and contract type to test whether the early-tenure churn risk is driven primarily by month-to-month customers, which would sharpen the targeting recommendation considerably.
