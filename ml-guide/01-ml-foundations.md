# 01 — Machine Learning Foundations

**Playlist videos 1–14.** This section builds the mental model everything else hangs on: what
machine learning actually *is*, the ways it's categorized, and the lifecycle of a real project.
No algorithm here yet — just the framing. Get this right and the rest of the guide is easier.

---

## 1. What is Machine Learning?

The classic definition (Arthur Samuel, 1959): *machine learning is the field of study that gives
computers the ability to learn without being explicitly programmed.* Tom Mitchell's sharper,
operational version is worth memorizing:

> A program learns from **experience E** with respect to some **task T** and **performance
> measure P**, if its performance on T, as measured by P, improves with experience E.

The key contrast is with **traditional programming**:

- **Traditional programming:** you write explicit rules. `input + rules → output`. A human
  encodes the logic ("if email contains 'lottery' and 'winner', mark spam").
- **Machine learning:** you supply examples of inputs *and their correct outputs*, and the machine
  infers the rules. `input + output → rules (a model)`. Then you apply those learned rules to new
  inputs.

Why this flip matters: for many problems the rules are impossible to write by hand. Nobody can
enumerate every pixel pattern that makes an image "a cat," or every phrasing that makes a review
"negative." But we *can* collect labeled examples. ML is the right tool precisely when (a) the
problem has patterns, (b) those patterns are hard to pin down as explicit rules, and (c) you have
data. If you can easily write the rules, you don't need ML — just write the rules.

---

## 2. AI vs ML vs DL

These are **nested**, not competing, concepts:

- **Artificial Intelligence (AI)** — the broadest umbrella: any technique that makes machines
  mimic human intelligence. This includes rule-based/expert systems (no learning at all), search
  algorithms, and ML.
- **Machine Learning (ML)** — a *subset* of AI: systems that learn patterns from data rather than
  being hand-coded.
- **Deep Learning (DL)** — a *subset* of ML that uses multi-layer neural networks. Its defining
  trait is **automatic feature extraction**: instead of a human engineering the input features,
  the network learns useful representations layer by layer.

The practical distinction between "classical ML" and DL:

| | Classical ML | Deep Learning |
|---|---|---|
| Feature engineering | Mostly manual (you decide the inputs) | Learned automatically |
| Data needed | Works with small/medium data | Hungry — usually needs lots of data |
| Compute | Modest | Heavy (often GPUs) |
| Interpretability | Often easier | Usually a black box |

This playlist (and this guide) is almost entirely **classical ML**. That's a feature: classical ML
is where you build the intuitions — bias/variance, regularization, evaluation — that carry over to
everything, including deep learning.

---

## 3. Types of Machine Learning

The primary taxonomy is by **what kind of supervision signal** the data carries.

### Supervised learning
The training data is **labeled**: every input `x` comes with the correct answer `y`. The model
learns the mapping `x → y`. Two sub-types:

- **Regression** — the target is a *continuous* number (house price, temperature, tomorrow's
  sales).
- **Classification** — the target is a *category* (spam/not-spam, digit 0–9, disease/no-disease).
  Binary = two classes; multiclass = more than two.

### Unsupervised learning
The data has **no labels** — just inputs. The goal is to find structure on its own:

- **Clustering** — group similar points (customer segments).
- **Dimensionality reduction** — compress many features into a few that retain the signal (PCA).
- **Association / anomaly detection** — find rules or outliers.

### Semi-supervised learning
A *little* labeled data and a *lot* of unlabeled data. Common in the real world because labeling is
expensive (think: millions of photos, only a few thousand hand-tagged). You leverage the structure
of the unlabeled data to make the small labeled set go further.

### Reinforcement learning (RL)
No dataset of correct answers. Instead an **agent** takes **actions** in an **environment**,
receives **rewards** or **penalties**, and learns a **policy** that maximizes cumulative reward
over time (game-playing bots, robotics, recommendation). It learns from *consequences*, not from a
labeled answer key.

---

## 4. Batch (Offline) vs Online Learning

This axis is about *how the model consumes data over time*.

### Batch / Offline learning
The model is trained **once**, on the entire available dataset, then deployed as a frozen artifact.
To incorporate new data you **retrain from scratch** (usually on a schedule — nightly, weekly).

- **Pros:** simple, stable, easy to reason about and test.
- **Cons:** can't adapt on the fly; retraining on huge data is expensive; the deployed model slowly
  goes stale as the world changes (**model rot / concept drift**).

### Online learning
The model is trained **incrementally**, one instance or one small **mini-batch** at a time, as data
streams in. Each new example nudges the model.

- **Pros:** adapts continuously; handles data that doesn't fit in memory (train on chunks and
  discard); good for streaming systems (stock prices, sensor feeds).
- **Cons:** sensitive to bad data — a burst of garbage can degrade the model fast, so it needs
  monitoring. The **learning rate** controls *how fast* it adapts: too high forgets the past too
  quickly, too low adapts too slowly.

A useful concept here is **out-of-core learning**: online-style training used specifically because
the dataset is too big for RAM — you stream it through in pieces.

> These aren't about "online" as in the internet. "Online" means *learning as data arrives*.

---

## 5. Instance-Based vs Model-Based Learning

This axis is about *how the system generalizes to new data*.

### Instance-based learning
The system **memorizes** the training examples and, at prediction time, compares a new point to the
stored ones using a **similarity measure**. There is no compact "learned function" — the data *is*
the model. Classic example: **K-Nearest Neighbors** — to classify a new point, look at the `k`
closest training points and take a majority vote.

- Training is trivial (just store the data); prediction is expensive (compare against everything).
- Doesn't generalize beyond what's near the stored examples.

### Model-based learning
The system builds an **explicit model** — a function with parameters — that summarizes the pattern
in the data. Once trained, it discards the data and keeps only the parameters. Prediction is just
plugging the input into the function. Example: **linear regression** learns a slope and intercept;
to predict, you evaluate the line.

- Training does the heavy lifting; prediction is cheap.
- Generalizes via the assumed functional form.

Most of this guide is model-based; KNN (Section 15) is the main instance-based example.

---

## 6. Challenges in Machine Learning

A realistic list of what actually goes wrong — mostly **data problems**, not algorithm problems.

- **Not enough data.** Most algorithms need a lot of examples to work well; small data leads to
  unreliable models.
- **Non-representative data (sampling bias).** If your training data doesn't reflect the population
  you'll deploy on, the model generalizes poorly. A famous cautionary tale is any poll that samples
  the wrong crowd.
- **Poor-quality data.** Errors, noise, missing values, and outliers. Real projects spend most time
  cleaning data (Sections 06–07 are entirely about this).
- **Irrelevant features.** "Garbage in, garbage out." Good models need good features — hence the
  large feature-engineering block in this playlist.
- **Overfitting.** The model memorizes the training data (including its noise) and fails on new
  data. It's too complex for the amount of signal available.
- **Underfitting.** The model is too simple to capture the real pattern — poor on both training and
  test data.
- **Software / engineering challenges.** Deployment, scaling, versioning, monitoring — ML in
  production is a systems problem, not just a modeling problem.

Overfitting vs underfitting is the single most important tension in ML; it gets a full formal
treatment as the **bias–variance trade-off** in Section 11.

---

## 7. Applications of Machine Learning

The point of this survey is to recognize the *shape* of ML problems in the wild:

- **Retail / e-commerce:** recommendation engines, demand forecasting, dynamic pricing.
- **Banking / finance:** fraud detection, credit scoring, algorithmic trading.
- **Transport:** ETA prediction, route optimization, self-driving perception.
- **Healthcare:** diagnosis from images, risk prediction, drug discovery.
- **Media:** content recommendation (YouTube, Netflix), search ranking.
- **NLP:** spam filtering, sentiment analysis, translation, chatbots.
- **Vision:** face recognition, object detection, medical imaging.

Notice each maps to a type from Section 3: recommendations ≈ a mix; fraud ≈ classification (often
with severe class imbalance — see Section 24); demand forecasting ≈ regression; customer
segmentation ≈ clustering.

---

## 8. Machine Learning Development Life Cycle (MLDLC)

ML projects follow a repeatable, **iterative** lifecycle. It is not linear — you loop back
constantly.

1. **Frame the problem.** Define the business goal, the ML task type, and the performance metric.
   (Expanded in Section 12 below.)
2. **Gather data.** From CSVs, databases, APIs, scraping (Section 02).
3. **Data preprocessing / cleaning.** Handle missing values, outliers, duplicates, wrong types
   (Sections 06–07).
4. **Exploratory Data Analysis (EDA).** Understand distributions and relationships (Section 03).
5. **Feature engineering & selection.** Create, transform, scale, encode features (Sections 04–08).
6. **Model training, evaluation & selection.** Try algorithms, tune, compare with proper metrics
   (most of the rest of the guide).
7. **Model deployment.** Ship it as an API/service.
8. **Testing & monitoring / optimization.** Watch for drift, retrain as needed — which loops you
   back to step 2.

The big lesson: **steps 2–5 (data + features) consume the majority of the effort**, and they
determine your ceiling far more than the choice of algorithm.

---

## 9. Data Job Roles

Because the terms get muddled, here's the honest separation of concerns:

- **Data Engineer** — builds and maintains the *data infrastructure*: pipelines, warehouses, ETL,
  ensuring clean data flows reliably. Heavy on SQL, big-data tools, systems.
- **Data Analyst** — answers *business questions* from existing data: dashboards, reports,
  descriptive statistics, visualization. Communicates insight to stakeholders.
- **Data Scientist** — builds *predictive models and experiments*: statistics + ML to answer "what
  will happen / why," often with messy, open-ended problems.
- **ML Engineer** — *productionizes* models: takes a model and makes it a scalable, monitored,
  reliable service. The intersection of software engineering and ML.

In small companies one person wears all these hats; in large ones they're distinct teams. The
pipeline roughly flows: Data Engineer → Analyst/Scientist → ML Engineer.

---

## 10. Tensors

A **tensor** is just an *n-dimensional array of numbers* — the universal container for data in ML.
The name comes from the number of axes (the **rank**):

- **0-D tensor (scalar):** a single number. `5`
- **1-D tensor (vector):** a list. `[1, 2, 3]` — e.g. one row of features.
- **2-D tensor (matrix):** rows × columns. A whole tabular dataset (samples × features).
- **3-D tensor:** e.g. a batch of sequences, or a single color image is often (height × width ×
  channels).
- **4-D tensor:** a *batch* of color images: (batch × height × width × channels).
- **5-D tensor:** e.g. video (batch × frames × height × width × channels).

Why care? Because every ML library represents data as tensors, and understanding the **shape** of
your data (how many axes, what each axis means) is a constant, practical necessity. Key vocabulary:
**rank/axes** (number of dimensions), **shape** (size along each axis).

---

## 11. Tooling: Anaconda, Jupyter, Colab

The standard classical-ML toolkit:

- **Anaconda** — a Python distribution that bundles the data-science stack (NumPy, pandas,
  scikit-learn, matplotlib) and manages environments (`conda`). Saves you from dependency hell.
- **Jupyter Notebook** — an interactive, cell-based environment where code, output, plots, and
  notes live together. Ideal for exploration because you run code piece by piece and see results
  immediately.
- **Google Colab** — Jupyter in the cloud, free, with optional GPU/TPU. Nothing to install; good
  for sharing and for compute you don't have locally.

The core libraries you'll use throughout:

- **NumPy** — fast numerical arrays (tensors).
- **pandas** — tabular data (DataFrames): loading, cleaning, EDA.
- **scikit-learn** — the classical-ML workhorse: consistent `fit` / `predict` / `transform` API for
  virtually every algorithm and preprocessing step in this guide.
- **matplotlib / seaborn** — visualization.

```python
# The scikit-learn API is uniform across almost everything you'll meet in this guide:
from sklearn.linear_model import LinearRegression
model = LinearRegression()
model.fit(X_train, y_train)      # learn parameters from data
preds = model.predict(X_test)    # apply learned model to new data
```

---

## 12. End-to-End Toy Project & Framing an ML Problem

Before diving into algorithms, the playlist walks through a full toy project end-to-end. The
learning outcome isn't the specific dataset — it's internalizing the MLDLC loop and, especially,
**how to frame a problem**, which is where projects succeed or fail before a line of model code is
written.

**Framing checklist:**

1. **What is the business objective?** ("Reduce churn," not "build a model.") The model is a means.
2. **What does the current solution look like?** A baseline (even a hand-rule) tells you if ML is
   worth it and gives a bar to beat.
3. **How do you frame it as an ML task?**
   - Supervised or unsupervised? (Do you have labels?)
   - Regression or classification? (Continuous or categorical target?)
   - Batch or online? (Section 4.)
4. **What is the performance metric?** Pick it *up front* and make sure it aligns with the business
   goal. For a fraud model, raw accuracy is a trap (Section 24); precision/recall matter (Section
   13). Choosing the wrong metric optimizes the wrong thing.
5. **What data do you have / need, and is it representative?** (Section 6's challenges.)
6. **What are the assumptions and constraints?** Latency budget, interpretability requirements,
   regulatory limits.

Getting the framing right determines everything downstream — the data you gather, the features you
build, the model you pick, and how you'll know if it worked.

---

## Key takeaways

- ML flips traditional programming: learn the rules from `(input, output)` examples instead of
  hand-coding them. Use it when patterns exist but rules are hard to write and you have data.
- AI ⊃ ML ⊃ DL. This guide is classical ML — the foundation for everything, including DL.
- Three axes to classify any ML system: **supervision** (supervised / unsupervised / semi / RL),
  **timing** (batch / online), and **generalization** (instance-based / model-based).
- Most real-world difficulty is **data**, not algorithms — and most project effort goes into the
  data/feature steps of the MLDLC.
- Frame the problem and pick the metric *before* modeling. The metric must reflect the business
  goal.

**Next:** [02 — Data Gathering](02-data-gathering.md) — actually getting the data in.
