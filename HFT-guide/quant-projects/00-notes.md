# Quant Research Profile — Notes

> Reasoning and reference material behind the project shortlist: the target job,
> my portfolio gap analysis, and the libraries to build *with*. The actual
> buildable projects live in `01-projects.md`. This file is context, not the project list.

## The job we're targeting

Trexquant / WorldQuant / Cubist-style **Quantitative Researcher**. What they screen for:

1. **Alpha / signal research** on large cross-sectional datasets → market-neutral factors.
2. **Order book / microstructure** modeling → short-term price impact, LOB dynamics, latency arbitrage.
3. **Statistical rigor** → hypothesis testing, avoiding overfitting, proper backtest methodology.

JD keeps repeating: *"ML techniques grounded in sound statistical reasoning, not just
generic algorithmic applications."* → the differentiator is **statistical validation**,
not more models.

## Honest gap analysis of my current portfolio

- Existing: [QuantStream-Analytics-Platform](https://github.com/MeetModi24/QuantStream-Analytics-Platform)
  (Java pipeline + FastAPI infra) and [Limit-Order-Book](https://github.com/MeetModi24/Limit-Order-Book)
  (microstructure/matching engine) + HFT C++ track.
- **Covered:** infrastructure + microstructure legs.
- **Thin:** the **alpha research + statistical validation** leg — which is exactly what
  alpha-research shops interview on. Prioritize that.

## Reference material (NOT projects — libraries/books to build ON TOP of)

These came up in chat as ingredients. They are **learn-from / build-with**, not portfolio
projects by themselves.

| Item | URL | Role | Note |
|------|-----|------|------|
| alphalens-reloaded | https://github.com/stefan-jansen/alphalens-reloaded | Factor eval (IC, quantiles, turnover) | ⭐653 · Apache-2.0 · maintained fork; original Quantopian repo is dead |
| machine-learning-for-trading | https://github.com/stefan-jansen/machine-learning-for-trading | End-to-end ML-quant codebook | ⭐21k · MIT · best single blueprint |
| WorldQuant_alpha101_code | https://github.com/yli188/WorldQuant_alpha101_code | 101 formulaic alphas + paper | ⭐870 · signal library to reimplement |
| DeepLOB | https://github.com/zcakhaa/DeepLOB-Deep-Convolutional-Neural-Networks-for-Limit-Order-Books | CNN+LSTM on LOB, FI-2010 data | ⭐608 · citable result |
| nautilus_trader | https://github.com/nautechsystems/nautilus_trader | Production Rust+Python backtest/live engine | ⭐29k · LGPL-3.0 · active |
| ABIDES (JPMorgan) | https://github.com/jpmorganchase/abides-jpmc-public | Agent-based LOB market simulator | ⭐174 · BSD-3 · **archived Jun 2025** (still works) |
| Microsoft Qlib | https://github.com/microsoft/qlib | Full AI-quant platform | ⭐~15k · MIT · alternative platform |

### ⚠️ The mlfinlab trap
`hudson-and-thames/mlfinlab` is linked everywhere for de Prado's **purged/embargoed CV,
triple-barrier labeling, meta-labeling, deflated Sharpe**. The public repo is now
"all rights reserved" and the real library is **paywalled**. Implement these yourself
from *Advances in Financial Machine Learning* — rolling your own purged CV is itself a
strong portfolio signal and is the exact "sound statistical reasoning" the JD wants.

## The research loop to demonstrate (what a project should SHOW)

```
data → signal/feature → statistical validation (purged CV, deflated Sharpe) →
microstructure-aware backtest → execution under latency → performance monitoring
```

Firms care that you understand **why standard k-fold CV is invalid on financial time
series**. Any project that skips validation loses the point.

## Indian HFT context (relevant to my search)

Target firms: **iRage, Tower Research (Mumbai), Graviton, Quadeye, AlphaGrep,
NK Securities, Dolat, Estee, WorldQuant India.** Stack pattern: **C++ for low-latency
execution, Python for research.** Markets: NSE/BSE, Nifty, Bank Nifty, F&O.
→ project list includes India-specific (NSE data, broker APIs) options.
