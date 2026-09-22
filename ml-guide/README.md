# Machine Learning — Theory Guide

A theory-first companion to CampusX's **"100 Days of Machine Learning"** playlist (134 videos).
The goal here is *understanding*, not code walkthroughs: what each idea means, the intuition
behind it, why it works, when it breaks, and how the pieces connect. Math appears only where it
genuinely clarifies the concept (the same places the videos slow down and derive it). Short code
snippets are included only when they make an idea concrete.

> This guide is organized so each file maps 1:1 to a contiguous block of playlist videos. The
> video number ranges are noted in each section so you can watch and read in parallel.

## How to read this

- **If you're new:** read top to bottom. The order is deliberate — data → features → models → evaluation → ensembles → unsupervised.
- **If you're revising:** jump to a section; each file is self-contained but cross-links related ideas.
- **Notation:** vectors/matrices in bold where it matters; otherwise plain. Loss functions, gradients, and probabilities are spelled out in words first, symbols second.

## Table of contents

| # | File | Videos | Topics |
|---|------|--------|--------|
| 01 | [ml-foundations.md](01-ml-foundations.md) | 1–14 | What ML is, AI/ML/DL, types of ML, batch vs online, instance vs model, challenges, applications, MLDLC, data roles, tensors, tooling, first project, framing a problem |
| 02 | [data-gathering.md](02-data-gathering.md) | 15–18 | CSV, JSON/SQL, APIs, web scraping |
| 03 | [eda.md](03-eda.md) | 19–22 | Understanding data, univariate, bivariate/multivariate analysis, profiling |
| 04 | [feature-engineering-scaling-encoding.md](04-feature-engineering-scaling-encoding.md) | 23–29 | Feature engineering, scaling, encoding, ColumnTransformer, pipelines |
| 05 | [feature-transforms.md](05-feature-transforms.md) | 30–34 | Function & power transforms, binning, mixed & datetime variables |
| 06 | [missing-data.md](06-missing-data.md) | 35–40 | CCA, simple/most-frequent imputation, missing indicators, KNN imputer, MICE |
| 07 | [outliers.md](07-outliers.md) | 41–44 | Outlier detection: z-score, IQR, percentile/winsorization |
| 08 | [dimensionality-pca.md](08-dimensionality-pca.md) | 45–49 | Feature construction, curse of dimensionality, PCA |
| 09 | [linear-regression.md](09-linear-regression.md) | 50–56 | Simple & multiple linear regression, regression metrics, assumptions |
| 10 | [gradient-descent.md](10-gradient-descent.md) | 57–60 | Gradient descent: batch, stochastic, mini-batch |
| 11 | [regularization.md](11-regularization.md) | 61–69 | Polynomial regression, bias–variance, Ridge, Lasso, ElasticNet |
| 12 | [logistic-regression.md](12-logistic-regression.md) | 70–75, 79–81 | Perceptron trick, sigmoid, MLE loss, softmax, hyperparameters |
| 13 | [classification-metrics.md](13-classification-metrics.md) | 76–78 | Confusion matrix, precision/recall/F1, ROC–AUC |
| 14 | [naive-bayes.md](14-naive-bayes.md) | 82–90 | Probability foundations, Bayes' theorem, Naive Bayes |
| 15 | [knn.md](15-knn.md) | 91 | K-Nearest Neighbors |
| 16 | [svm.md](16-svm.md) | 92–96 | SVM geometric intuition, hard/soft margin, kernel trick |
| 17 | [decision-trees.md](17-decision-trees.md) | 97–100 | Entropy/Gini/information gain, hyperparameters, regression trees |
| 18 | [ensembles-voting-bagging.md](18-ensembles-voting-bagging.md) | 101–107 | Ensemble learning, voting, bagging |
| 19 | [random-forest.md](19-random-forest.md) | 108–114 | Random forest, bias–variance, hyperparameters, OOB, feature importance |
| 20 | [adaboost.md](20-adaboost.md) | 115–119 | AdaBoost, bagging vs boosting |
| 21 | [gradient-boosting-xgboost.md](21-gradient-boosting-xgboost.md) | 120–126 | Gradient boosting, XGBoost (regression, classification, math) |
| 22 | [stacking-blending.md](22-stacking-blending.md) | 127 | Stacking and blending |
| 23 | [clustering.md](23-clustering.md) | 128–132 | K-Means, hierarchical, DBSCAN |
| 24 | [imbalanced-data.md](24-imbalanced-data.md) | 133 | Undersampling, oversampling, SMOTE |
| 25 | [hyperparameter-tuning.md](25-hyperparameter-tuning.md) | 134 | Optuna, Bayesian optimization |

## Status

This guide is being written section by section. Completed sections are linked above; if a file
isn't present yet, it's still being written.
