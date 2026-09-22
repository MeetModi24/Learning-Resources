# HFT Guide — C++ Mastery + Infrastructure

An end-to-end guide for an HFT / quant-developer interview: first the **C++ language and systems
surface** an HFT interview probes (learncpp/Cherno style — plain explanations, focused snippets,
tradeoffs, and the "why" behind each rule), then the **HFT infrastructure** that C++ gets used to
build — the market it lives in, the feed handler, and the limit order book & matching engine.

The guide is split into two parts, each in its own folder:

- **`cpp-guide/`** — C++ from the memory model up to lock-free systems programming (Modules 1–18).
- **`hft-infrastructure-guide/`** — the trading-systems domain and the components you build, with its
  own 1-based numbering (Modules 01→02→03…).

## How to read this

Go top-to-bottom. Each C++ module: concept → snippet → tradeoffs → interview "why" → how it appears in
the order book. Snippets are illustrative (they show one idea), not full programs — but they're
written to be correct so you can drop them into a file and compile with `-std=c++20`. The
infrastructure modules add the domain context and the systems design those C++ skills feed into.

---

## Part A — C++ mastery (`cpp-guide/`)

### Part 0 — Foundations (the memory mental model)

1. [Memory & object lifetime: stack vs heap](cpp-guide/01-memory-lifetime.md)
2. [Pointers & references](cpp-guide/02-pointers-references.md)
3. [Dynamic memory (`new`/`delete`) & why the heap is slow](cpp-guide/03-dynamic-memory.md)
4. [lvalues, rvalues & an intro to moving](cpp-guide/04-value-categories.md)

### Part I — Language mastery

5. [`const`, `constexpr`, `consteval`, const-correctness](cpp-guide/05-const-constexpr.md)
6. [Classes: constructors, destructors, `this`, access control](cpp-guide/06-classes.md)
7. [RAII & the Rule of 0/3/5](cpp-guide/07-raii-rule-of-five.md)
8. [Operator overloading & value semantics](cpp-guide/08-operator-overloading.md)
9. [Move semantics & rvalue references (deep)](cpp-guide/09-move-semantics.md)
10. [Templates → Concepts](cpp-guide/10-templates-concepts.md)
11. [Inheritance, virtual functions, vtables — and when not to use them](cpp-guide/11-inheritance-virtual.md)
12. [The STL: containers, iterators, algorithms, complexity & cache](cpp-guide/12-stl.md)
13. [Smart pointers & ownership](cpp-guide/13-smart-pointers.md)
14. [Exceptions, `noexcept`, error handling without exceptions](cpp-guide/14-exceptions-errors.md)

### Part II — Systems / HFT-specific C++

15. [Memory model, cache, alignment, false sharing](cpp-guide/15-memory-cache.md)
16. [`std::atomic`, memory ordering, lock-free basics](cpp-guide/16-atomics-lockfree.md)
17. [Zero-cost abstraction, CRTP, branch elimination](cpp-guide/17-zero-cost-crtp.md)
18. [C++20 features that matter: concepts, `<bit>`, `std::span`, ranges](cpp-guide/18-cpp20-features.md)

---

## Part B — HFT infrastructure (`hft-infrastructure-guide/`)

Its own numbering, starting at 01. The modules follow the data's journey through the pipeline: the
big picture first, then the feed handler that fills the book, then the book itself, then the strategy
engine that decides what to trade, then the OMS that fires those orders, then the backtester &
simulator that validate the whole thing, then (planned) the options greeks engine.

1. [Market microstructure & the HFT system pipeline (India-first)](hft-infrastructure-guide/01-market-microstructure-and-hft-systems.md)
2. [The feed handler in C++ (deep dive)](hft-infrastructure-guide/02-feed-handler.md)
3. [Order-book architecture: building it with everything above](hft-infrastructure-guide/03-order-book-project.md)
4. [The strategy engine: knowhow + design problems (LLD & HLD)](hft-infrastructure-guide/04-strategy-engine.md)
5. [The OMS / order gateway: core concepts + design problems (brute→HFT)](hft-infrastructure-guide/05-oms-order-gateway.md)
6. [The backtester & the exchange simulator: core concepts + design problems](hft-infrastructure-guide/06-backtester-and-simulator.md)
7. [The five systems at a glance: capstone cheat-sheet (reqs, challenges, flow, HFT techniques)](hft-infrastructure-guide/07-systems-at-a-glance.md)
8. Options pricing / greeks engine *(planned)*

---

## The mental through-line

> Objects live somewhere (stack/heap) for some lifetime. You refer to them via pointers/references.
> Ownership decides who frees what (RAII/smart pointers). Templates give abstraction with zero runtime
> cost. The hardware (cache, atomics) decides how fast it actually runs. Then the market: data leaves
> the exchange as a fast lossy feed, a feed handler makes it trustworthy, and the order book is where
> all of the C++ meets a real problem.
