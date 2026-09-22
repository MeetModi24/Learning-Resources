# Buildable Quant / HFT Portfolio Projects

> Researched, URL-verified (GitHub API: stars, license, last-push, archived status as of
> Sep 2026). These are **projects you clone and EXTEND** into original work — not lectures,
> papers, or "awesome" lists. Each entry says what to *build*, not just what to run.
> Difficulty is honest. Reference libraries live in `00-notes.md`.

Legend: 🟢 beginner · 🟡 medium · 🔴 advanced · ⭐ = stars

---

## How to read this

Cloning a repo and running its README is **not** a portfolio. Every entry below has an
**"Extend into"** line — that's the actual project. The differentiator for an
alpha-research / HFT job is not the base repo, it's the original engineering + statistical
validation you layer on top. Pick **one global project + one India-specific project** and
go deep, rather than collecting clones.

---

## TIER 1 — Start here (best signal-to-effort, medium difficulty)

### 1. Optiver Ready-Trader-Go market maker 🟡
- **Repo:** https://github.com/keanekwa/Optiver-Ready-Trader-Go — ⭐105 · GPL-3.0 · Python
- **What:** Avellaneda–Stoikov market maker built on Optiver's *actual* recruiting-competition simulator.
- **Why it's #1:** It is literally shaped like a quant-firm take-home. Inventory-risk-aware
  market making is exactly what Cubist/Trexquant/Optiver interviews probe.
- **Extend into:** replace the static A–S parameters with an online-estimated volatility +
  inventory-skew model; stress-test against adversarial order flow; report PnL vs. inventory
  risk curves. → *your project: "adaptive market maker under adversarial flow."*

### 2. Extend your own Limit-Order-Book with a LIVE NSE depth feed 🟡→🔴 ⭐ HIGHEST DIFFERENTIATION
- **Base:** your [Limit-Order-Book](https://github.com/MeetModi24/Limit-Order-Book) + Kite WebSocket.
- **Finding:** research turned up **no mature Indian-market order-book/microstructure repo.**
  That gap *is* the opportunity. Kite/Upstox/AngelOne WebSockets send **5-level market depth**.
- **Closest starter to borrow from:** https://github.com/rthennan/ZerodhaWebsocket — ⭐35 · Python ·
  active (May 2026) — captures tick-by-tick for Nifty 500 + F&O into a DB.
- **Extend into:** a C++ LOB **reconstructor** fed by live NSE depth-5 WebSocket ticks (instead
  of synthetic data), persisting an actual order-book state machine + computing order-flow
  imbalance in real time. Nobody has published this for NSE → hard to fake, maps directly to
  the C++ low-latency skill iRage/Tower/Graviton screen for.

### 3. DeepLOB reimplementation + architecture bake-off 🟡
- **Repo (canonical):** https://github.com/zcakhaa/DeepLOB-Deep-Convolutional-Neural-Networks-for-Limit-Order-Books
  — ⭐608 · no license file · stale (2021) but authors' own repo · ships FI-2010 data.
- **Cleaner modern base:** https://github.com/Jeonghwan-Cheon/lob-deep-learning — ⭐160 · Python · active.
- **What:** CNN+LSTM predicting short-term price moves from the limit order book.
- **Extend into:** a **rigorous head-to-head** — DeepLOB vs. TABL vs. a plain gradient-boosted
  order-flow-imbalance model on the *same* FI-2010 split, with proper stats (not just accuracy).
  → the "rigorous comparison, sound statistical reasoning" narrative the JD demands.

---

## TIER 2 — LOB / Matching-Engine Simulators (C++, systems signal)

### 4. liquibook 🟡
- https://github.com/enewhuis/liquibook — ⭐1,507 · "Other" license · C++ · last push Mar 2024
- Price/time-priority matching engine, depth-of-book. The most-starred clean C++ engine.
- **Extend into:** wrap it with an ITCH/FIX feed handler + a latency-instrumented order-entry
  gateway, then run your own strategies against it and publish tick-to-trade latency percentiles.

### 5. DistributedATS 🔴
- https://github.com/mkipnis/DistributedATS — ⭐120 · "Other" · C++ · **active (2026)**
- Multi-matching-engine exchange (CLOB) over QuickFIX + Liquibook + DDS.
- **Extend into:** add a market-data replay module from real tick data; benchmark matching
  latency under load. Shows distributed-exchange architecture + FIX connectivity.

### 6. LimitOrderBook-MatchingEngine 🟡 / lob_sim 🟢
- https://github.com/jxm35/LimitOrderBook-MatchingEngine — ⭐32 · C++ · active — LOB + matching + viz.
- https://github.com/Blaeriz/lob_sim — ⭐2 · C · red-black-tree levels, O(1) cancel, ~258k ticks/sec.
- **Extend into:** feed real L2/L3 (LOBSTER) data + overlay order-flow-imbalance signals; or
  port price-level storage to an intrusive skip-list/radix tree and publish a latency writeup.
- **Read (don't clone) reference:** `exchange-core/exchange-core` — ⭐2,633 · Apache-2.0 · Java ·
  stale but gold-standard LMAX-Disruptor matching architecture.

---

## TIER 3 — Backtesting engines (build a strategy ON TOP)

### 7. nautilus_trader 🟡→🔴 — the production-grade pick
- https://github.com/nautechsystems/nautilus_trader — ⭐29,263 · LGPL-3.0 · Rust+Python · **active**
- Event-driven, nanosecond, order-book-aware backtest + live. Not a toy.
- **Extend into:** implement an OFI-based microstructure `Strategy`, backtest on L2 data, then
  deploy to a paper-trading adapter → covers strategy dev + backtest + low-latency deploy in one.

### 8. Lighter entry points
- https://github.com/mementum/backtrader — ⭐23,303 · GPL-3.0 · Python · best-documented starter.
- https://github.com/kernc/backtesting.py — ⭐8,982 · AGPL-3.0 · Python · quick vectorized MVP.
- https://github.com/QuantConnect/Lean — ⭐21,721 · Apache-2.0 · C# (Python via lean-cli) · institutional-grade.
- **Extend into:** custom OFI `Indicator`; add walk-forward optimization + `alphalens` factor-decay;
  or a stat-arb pairs strategy paper-traded through IBKR/Alpaca for a "live" proof-point.

---

## TIER 4 — Alpha / factor research pipelines

### 9. Microsoft Qlib 🟡→🔴
- https://github.com/microsoft/qlib — ⭐48,749 · MIT · Python(+C++) · **active (pushed today)**
- Full research pipeline: features → factor models → backtest → portfolio.
- **Extend into:** a custom Alpha158-style factor set built from **LOB-derived features**;
  benchmark IC / rank-IC against Qlib baselines.

### 10. machine-learning-for-trading 🟡
- https://github.com/stefan-jansen/machine-learning-for-trading — ⭐20,972 · MIT · runnable pipelines per chapter.
- **Extend into:** take the LOB/HFT chapter, rebuild with your own tick data + a gradient-boosted
  or transformer signal model.
- **Pair with:** `quantopian/alphalens` (⭐4,450, Apache-2.0) for IC/quantile/turnover rigor, and
  `yli188/WorldQuant_alpha101_code` (⭐870, no license — check before reuse) as the signal library
  to reimplement in a vectorized/Polars pipeline and rank by IC decay.

---

## TIER 5 — Statistical arbitrage / pairs trading (thin field — plan to build more yourself)

### 11. Statistical-Arbitrage 🟡
- https://github.com/bradleyboyuyang/Statistical-Arbitrage — ⭐282 · MIT · Jupyter · (2023)
- Cointegration-based high-frequency stat arb — directly on the JD.
- **Extend into:** cross-exchange crypto pair + Kalman-filtered (regime-adaptive) hedge ratio
  instead of static OLS; add a transaction-cost model + rolling half-life stop.

### 12. Trading-System (C++ async) 🔴
- https://github.com/bradleyboyuyang/Trading-System — ⭐69 · Apache-2.0 · C++ · (Mar 2024)
- Low-latency async trading system — the execution layer for a stat-arb strategy.
- **Extend into:** bridge your Python pairs signal → this C++ executor via gRPC/IPC → a full
  research-to-execution stack story. (Also the best "build-your-own-system" C++ base found.)

> Candid: pure pairs-trading repos are mostly 0–3★ student projects. The two above + de Prado's
> methodology (purged CV, deflated Sharpe — implement yourself, see `00-notes.md`) are the play.

---

## TIER 6 — INDIA-SPECIFIC (NSE/BSE, for iRage / Tower Mumbai / Graviton / Quadeye / AlphaGrep)

### 13. OpenAlgo — the standout India platform 🟡 ⭐ BEST INDIA ROI
- https://github.com/marketcalls/openalgo — ⭐2,711 · AGPL-3.0 · Python · **very active**
- Full self-hosted algo platform: Flask+React, **36 broker integrations** (Kite/Upstox/AngelOne/
  Breeze…), REST API, options analytics, TradingView webhook bridge, ZeroMQ streaming, DuckDB tick store.
  This is the one India repo big enough that "I extended OpenAlgo with X" reads as real engineering.
- **Client SDK:** https://github.com/marketcalls/openalgo-python-library (⭐47, MIT) — build strategies
  against the server without forking the whole platform.
- **Extend into:** add a **market-making / quote-simulation sandbox** against paper orders, OR a
  real-time NIFTY/BANKNIFTY **Greeks + IV-surface dashboard** from live depth data, OR optimize the
  tick-ingestion path for latency.

### 14. Free NSE/BSE data feeds (data-pipeline project) 🟢→🟡
- https://github.com/jugaad-py/jugaad-data — ⭐577 · Python · **very active (today)** — bhavcopy,
  indices, F&O EOD, no API key. Best free historical source.
- https://github.com/aeron7/nsepython — ⭐367 · GPL-3.0 · **active (2026)** — the live `nsepy` successor,
  patched against NSE anti-scraping.
- **⚠️ Dead:** `swapniljariwala/nsepy` (⭐808) — stale since Dec 2023, NSE keeps breaking it. Legacy only.
- **Extend into:** wire `nsepython`/`jugaad-data` as a **custom data feed into vanilla backtrader/zipline**
  — no clean published version exists (a real gap). Do it *properly*: survivorship-bias handling +
  F&O contract-rollover logic + document the NSE data quirks you had to fix.

### 15. Broker SDKs (infrastructure to build on, not projects themselves)
- https://github.com/zerodha/pykiteconnect — ⭐~1.3k · MIT · official Kite Connect (orders, WebSocket ticks).
- https://github.com/Upstox/upstox-python — ⭐189 · MIT · official.
- https://github.com/angel-one/smartapi-python — ⭐178 · official AngelOne.
- https://github.com/Idirect-Tech/Breeze-Python-SDK — ⭐88 · MIT · official ICICI Breeze
  (this is the *real* repo behind the `breeze-connect` PyPI package).

### 16. India strategy skeletons — GUT AND REBUILD, don't present as-is 🟢
- https://github.com/srikar-kodakandla/fully-automated-nifty-options-trading — ⭐249 · MIT — supertrend+ADX
  option-spread bot via Selenium; real end-to-end signal→execution example.
- https://github.com/buzzsubash/algo_trading_strategies_india — ⭐73 · active (2026) — short straddle/strangle
  on NIFTY/BANKNIFTY/SENSEX.
- **Candid:** these are weekend-project skeletons. Fine to learn strategy structure; not defensible in an
  interview without a full rewrite (proper backtest validation, risk controls, live/paper split).
- **⚠️ Overrated:** `uberdeveloper/fastbt` (⭐28) is **not** an official Zerodha project despite the
  association job posts imply — genuinely minimal EOD tool.

### 17. Regime-adaptive cross-sectional NSE alpha system 🟡→🔴 ⭐ DIRECT QUANT-RESEARCH FIT
- **Closest complete project:**
  https://github.com/shubham21-ai/Regime-Adaptive-Cross-Sectional-Alpha-Model-for-NSE-Equities
  — ⭐0 · **no license** · Python · active (Apr 2026).
- **What:** 291 NSE stocks, 11 daily features, LightGBM + iTransformer cross-sectional forecasts,
  a three-state HMM regime filter, walk-forward evaluation, Half-Kelly sizing, VaR/CVaR controls,
  ARIMA/SARIMA/VAR comparisons, and a Streamlit research dashboard. The repo reports 0.0223 mean
  daily Spearman IC and 0.228 ICIR over 467 validation days.
- **Why it is relevant:** this directly fills the portfolio's alpha-research gap: panel-data feature
  engineering, cross-sectional ranking, regime conditioning, out-of-sample measurement, portfolio
  construction, and risk attribution. It is a **systematic-equities / medium-frequency quant project**,
  not an HFT or low-latency project; pair it with the LOB/C++ work rather than describing it as HFT.
- **Candid result check:** the repo's own table reports 15.65% annual return and 0.74 Sharpe versus
  18.85% and 0.856 for Nifty 500. Its strongest evidence is positive OOS IC and lower drawdown
  (10.82% vs 15.80%), not benchmark-beating returns. Its reported VAR mean IC (+0.027) also exceeds
  iTransformer (+0.022), so the deep model has not clearly dominated every baseline.
- **Audit before using the resume claims:** verify point-in-time Nifty membership/delistings
  (survivorship bias), corporate-action handling, train-only scaling, causal HMM filtering rather than
  full-sample Viterbi labels, exact purge/embargo logic, ICIR convention, turnover/capacity, and Indian
  costs (brokerage, STT, exchange fees, impact). Add confidence intervals or a block bootstrap for IC
  and compare against simple momentum, mean-reversion, linear/ridge, CatBoost, and XGBoost baselines.
- **Extend into:** rebuild the universe from point-in-time NSE constituent snapshots; use sector- and
  beta-neutral portfolio construction; report IC decay by horizon/regime/sector; add realistic costs
  and liquidity constraints; then run ablations for features, HMM gate, model, and sizing. This turns a
  broad course project into defensible research.

#### Open-source comparisons and building blocks for #17

| Repository | Why it is useful | How to use it |
|---|---|---|
| [microsoft/qlib](https://github.com/microsoft/qlib) — ⭐48,752 · MIT | Mature cross-sectional research workflow with LightGBM, Alpha158/360, IC/Rank-IC/ICIR, model comparisons, and cost-aware portfolio backtests | Use as the evaluation contract; port the NSE panel into Qlib and reproduce every headline metric |
| [Caie777/end-to-end-stock-transformer](https://github.com/Caie777/end-to-end-stock-transformer) — ⭐13 · MIT | Closest research analogue: Transformer vs LightGBM for cross-sectional A-share returns with rolling OOS tests, IC/Rank-IC, turnover, seeds, and robustness notes | Borrow its experiment/config discipline and multi-seed reporting; do not copy its China-specific data assumptions |
| [thuml/iTransformer](https://github.com/thuml/iTransformer) — ⭐2,216 · MIT | Official iTransformer implementation and paper code | Use to validate that the model architecture is faithful; finance-specific labeling and leakage controls remain your responsibility |
| [Vraj3005/NF-LRD](https://github.com/Vraj3005/NF-LRD) — ⭐0 · no license | Nifty 50 HMM/GMM/MSR regime research with walk-forward testing, cost/slippage assumptions, Monte Carlo risk, tests, and Streamlit | Read as an India-specific regime/dashboard comparison; reimplement ideas because the repo has no reuse license |
| [Jevik-R/portfolio-ai](https://github.com/Jevik-R/portfolio-ai) — ⭐0 · MIT | NSE pipeline with CVaR/Black-Litterman allocation, walk-forward backtesting, macro overlay, and Streamlit | Reference for portfolio/risk UI structure; its 16-stock universe and sentiment signal are much narrower than #17 |
| [skfolio/skfolio](https://github.com/skfolio/skfolio) — ⭐2,428 · BSD-3-Clause | Production-quality portfolio optimization with walk-forward and combinatorial-purged CV, CVaR, transaction-cost, turnover, and weight constraints | Replace hand-rolled risk allocation where appropriate and benchmark Half-Kelly against constrained CVaR/risk-budgeting portfolios |

> License note: “public on GitHub” does not automatically permit copying. The Project Aegis and
> NF-LRD repositories had no detected license when checked; study them, but do not reuse their code
> without permission. Qlib, iTransformer, the A-share Transformer, PortfolioAI, and skfolio declare
> permissive licenses as listed above.

---

## Recommended path (do NOT collect all of these)

1. **Global microstructure/ML piece:** DeepLOB bake-off (#3) → feeds naturally into your existing repos.
2. **Global systems piece:** Optiver Ready-Trader-Go (#1) *or* extend liquibook (#4) for the C++ latency story.
3. **India differentiator:** live-NSE-depth LOB reconstructor (#2) — the single most original, hard-to-fake
   piece for an Indian HFT application, and it reuses your Limit-Order-Book repo.
4. **India platform credibility:** extend OpenAlgo (#13) with one real module.
5. **Alpha-research leg (closes your biggest gap):** rebuild the NSE cross-sectional system (#17) on
   Qlib (#9), with point-in-time membership, purged walk-forward validation, neutralization, IC decay,
   realistic costs, and multi-seed/model ablations.

Write each up as a `.md` (like your HFT-CPP-Portfolio-Research.md), emphasizing **statistical validation
and the research loop**, not just the code.
