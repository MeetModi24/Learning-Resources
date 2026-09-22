# C++ HFT Portfolio Projects: Comprehensive Research Report

## 🎯 Executive Summary

Breaking into HFT firms (Jane Street, Citadel, Jump Trading, HRT) requires demonstrating mastery across three dimensions: low-level systems programming, financial domain knowledge, and measurable performance optimization. The projects below are ranked by portfolio impact and learning value.

---

## 🏆 Tier 1: Must-Have Portfolio Projects

### 1. CppTrader ⭐⭐⭐⭐⭐

**GitHub:** [chronoxor/CppTrader](https://github.com/chronoxor/CppTrader)  
**Performance:** 41M msg/sec NASDAQ ITCH parsing (24ns latency), 102ns matching engine

#### What You Learn:
- Real NASDAQ ITCH protocol handling (processes 283M actual market messages)
- Three optimization levels showing concrete tradeoffs (309ns → 102ns)
- Cache-friendly sorted arrays vs Red-Black trees
- Object pooling and O(1) allocation strategies

#### Why It's Important
Shows you understand progressive optimization - not just making something fast, but knowing what to sacrifice for speed. The three-level implementation (standard/optimized/aggressive) demonstrates engineering judgment.

#### Portfolio Value
**Exceptional.** Includes real market data validation, comprehensive benchmarks, cross-platform builds. This alone can carry an HFT interview.

---

### 2. rigtorp/SPSCQueue + max0x7ba/atomic_queue ⭐⭐⭐⭐⭐

**GitHub:**
- [rigtorp/SPSCQueue](https://github.com/rigtorp/SPSCQueue)
- [max0x7ba/atomic_queue](https://github.com/max0x7ba/atomic_queue)

**Performance:** SPSCQueue: 362K ops/ms (133ns RTT), atomic_queue: <100ns RTT

#### What You Learn:
- Cache-line alignment and false sharing elimination
- Memory ordering semantics (acquire/release)
- Local index caching to reduce coherency traffic
- Cross-core latency penalties (3× slower)

#### Why It's Important
Lock-free programming is table stakes for HFT. Every firm asks about it. These projects teach the why behind memory barriers and cache coherency.

#### Portfolio Value
**Critical.** Fork these, add your own optimizations, benchmark on your hardware. Document what you learned about CPU architecture.

---

### 3. mansoor-mamnoon/limit-order-book ⭐⭐⭐⭐

**GitHub:** [mansoor-mamnoon/limit-order-book](https://github.com/mansoor-mamnoon/limit-order-book)  
**Performance:** 20.7M msg/sec, 42ns P50 latency

#### What You Learn:
- Slab allocator with O(1) alloc/free
- Side-specialized templates (compile-time branch elimination)
- Real Binance data capture and replay
- Complete backtesting framework (TWAP/VWAP/POV strategies)
- Market microstructure analytics (volatility, impact curves)

#### Why It's Important
End-to-end HFT pipeline from data capture → strategy → analytics. Shows you understand the full trading lifecycle, not just fast code.

#### Portfolio Value
**Very High.** The backtesting + analytics demonstrate quant skills alongside systems programming.

---

### 4. Anish Prakash's 27M Orders/sec Order Book ⭐⭐⭐⭐

**Source:** [Medium Article - "Building a 27M Orders/sec Limit Order Book from Scratch in C++20"](https://medium.com/@anishprakash/building-a-27m-orders-sec-limit-order-book-from-scratch-in-c-20-8d3f39e1935e)

#### What You Learn:
- Intrusive linked lists with 32-bit indices
- Bitset price level tracking with hardware instructions (ctz/clz)
- Zero-copy binary parsing
- Why std:: containers don't work at HFT scales

#### Why It's Important
Expert-level deep dive into memory layout and cache optimization. Written by a practitioner who explains the reasoning behind each choice.

#### Portfolio Value
**High.** Study this, then implement your own version. Reference it in interviews to show you've learned from industry experts.

---

## 🔧 Tier 2: Advanced Learning Projects

### 5. krish567366/submicro-execution-engine ⭐⭐⭐⭐

**GitHub:** [krish567366/submicro-execution-engine](https://github.com/krish567366/submicro-execution-engine)  
**Performance:** 890ns end-to-end decision latency (research-grade)

#### What You Learn:
- Hawkes processes and market microstructure models
- Avellaneda-Stoikov market making
- SIMD AVX-512 optimization (40ns signal extraction)
- Simulated DPDK/FPGA (educational, not production)

#### Why It's Important
Bridges academic theory (Kyle's lambda, flow toxicity) with systems implementation. Shows you understand the math behind HFT strategies.

#### Portfolio Value
**Medium-High.** Clearly marked as research/education. Good for demonstrating quantitative finance knowledge alongside C++ skills.

---

### 6. hft-infra-lab (Research Project) ⭐⭐⭐⭐

**Performance:** 17.8M orders/sec, 60M msg/sec ITCH parsing

#### What You Learn:
- Production-grade L3 matching engine (10 order types)
- DPDK kernel bypass (19.9M pkt/sec)
- Complete FIX 4.2 parser (5.5M msg/sec)
- 67 technical indicators (Ehlers, adaptive MAs, VWAP/TWAP)
- Market microstructure analytics (VPIN, Kyle's lambda, OFI)

#### Why It's Important
Comprehensive HFT infrastructure - protocols, matching, analytics, execution. Shows breadth across the entire stack.

#### Portfolio Value
**Very High.** This is a portfolio system, not just a component. Demonstrates you can architect end-to-end solutions.

---

### 7. Liquibook ⭐⭐⭐

**GitHub:** [enewhuis/liquibook](https://github.com/enewhuis/liquibook)  
**Performance:** 2-2.5M inserts/sec

#### What You Learn:
- Header-only modern C++ design
- Order lifecycle management
- Depth book dynamics and market data generation
- FAST protocol integration

#### Why It's Important
Production-quality reference implementation. Clean, well-documented, actively maintained. Good for understanding exchange architecture.

#### Portfolio Value
**Medium-High.** Fork it, extend it (add new order types, optimize hot paths), benchmark against CppTrader.

---

## 📚 Tier 3: Foundational Learning Resources

### 8. rigtorp/awesome-lockfree ⭐⭐⭐⭐⭐

**GitHub:** [rigtorp/awesome-lockfree](https://github.com/rigtorp/awesome-lockfree)

#### What You Learn:
- Michael-Scott Queue paper
- Herb Sutter's "Lock-Free Programming" talks
- Paul McKenney's memory barriers guide
- LMAX Disruptor patterns (Martin Thompson)
- 1024cores (Dmitry Vyukov's algorithms)

#### Why It's Important
Roadmap for self-study. These are the papers/talks that actual HFT engineers learned from.

#### Portfolio Value
**Essential.** Reference these in interviews ("I learned about X from the Michael-Scott paper..."). Shows you've done the deep reading.

---

### 9. Erik Rigtorp's Low Latency Tuning Guide ⭐⭐⭐⭐⭐

**Website:** [rigtorp.se](https://rigtorp.se/low-latency-guide/)

#### What You Learn:
- CPU isolation (isolcpus, nohz_full, rcu_nocbs)
- Disabling hyper-threading (doubles L1/L2 cache per thread)
- TLB shootdown avoidance
- Kernel bypass (DPDK, OpenOnload, Mellanox VMA)
- System jitter reduction (18μs vs 17ms)

#### Why It's Important
Shows you understand full-stack optimization - hardware, OS, application. HFT is systems engineering, not just algorithms.

#### Portfolio Value
**Critical.** Apply these techniques to your projects. Document before/after benchmarks.

---

### 10. Building Low Latency Applications with C++ (Sourav Ghosh) ⭐⭐⭐⭐

**Book:** [Packt Publishing](https://www.packtpub.com/product/building-low-latency-applications-with-c/9781837639359)

#### What You Learn:
- CPU architecture → C++20 implementation → trading strategies
- Which C++ features work at HFT scales
- GCC optimization, performance measurement
- Order management, market data handling
- Practical insights from 10+ years prop trading

#### Why It's Important
Practitioner-focused book from someone who built production HFT systems. Bridges theory and practice.

#### Portfolio Value
**High.** Work through the examples, implement the concepts in your own projects.

---

## 🚀 Additional High-Value Projects

### 11. Helix (Market Data Feed Handler) ⭐⭐⭐

**Performance:** NASDAQ TotalView-ITCH 5.0, MoldUDP, Parity PMD

#### What You Learn:
- Multi-protocol market data normalization
- Synthetic NBBO generation
- Feed aggregation architecture

#### Why It's Important
Understanding market data infrastructure is crucial. Most HFT systems have separate feed handlers.

---

### 12. cameron314/concurrentqueue ⭐⭐⭐⭐

**GitHub:** [cameron314/concurrentqueue](https://github.com/cameron314/concurrentqueue)  
**Performance:** Faster than boost and TBB under contention

#### What You Learn:
- MPMC lock-free queue (vs SPSC)
- Contiguous block storage (not linked lists)
- Sub-queue per producer for locality
- Formally verified with CDSChecker and Relacy

#### Why It's Important
Production-grade implementation used in real systems. Study alongside SPSCQueue to understand SPSC vs MPMC tradeoffs.

---

## 🎓 Learning Roadmap

### Phase 1: Foundations (2-3 months)

1. **LearnCpp.com** - C++11-C++23 fundamentals (skip if you know C++)
2. **awesome-lockfree** - Read the core papers (Michael-Scott Queue, memory model)
3. **SPSCQueue** - Fork, benchmark, document findings

### Phase 2: Core Skills (3-4 months)

1. **CppTrader** - Study all three optimization levels, benchmark on your hardware
2. **mansoor-mamnoon/limit-order-book** - Implement your own version, add features
3. **Anish Prakash's article** - Deep dive into cache optimization
4. **Rigtorp's tuning guide** - Apply to your projects

### Phase 3: Advanced (2-3 months)

1. **hft-infra-lab or Liquibook** - Full system implementation
2. **atomic_queue** - Study MPMC patterns
3. **submicro-execution-engine** - Learn market microstructure models

### Phase 4: Specialization (2-3 months)

1. **DPDK** - Official documentation, testpmd
2. **FIX protocol** - Implement parser from scratch
3. **Real market data** - Capture and replay (Binance WebSocket, NASDAQ ITCH)

---

## 💼 Portfolio Strategy

### For Your GitHub Profile:

1. **One comprehensive project** - Fork CppTrader or hft-infra-lab, add features
2. **One from-scratch implementation** - Order book engine
3. **Lock-free data structures** - SPSC/MPMC queues with benchmarks
4. **Documentation** - README with architecture diagrams, benchmark methodology, lessons learned

### For Interviews:

- **Talk about tradeoffs:** "I chose sorted arrays over Red-Black trees because..."
- **Show measurements:** "I reduced latency from X to Y by..."
- **Reference experts:** "Following Erik Rigtorp's tuning guide, I..."
- **Demonstrate breadth:** Systems (Linux tuning), algorithms (matching engine), domain (market microstructure)

---

## 🔥 Key Concepts to Master

### 1. Lock-Free Programming
- Memory ordering
- ABA problem
- CAS loops

### 2. Cache Optimization
- False sharing
- Cache-line alignment
- Prefetching

### 3. Memory Management
- Slab allocators
- Object pools
- Zero-copy

### 4. Kernel Bypass
- DPDK
- Huge pages
- CPU pinning

### 5. Market Data
- ITCH protocol
- FIX protocol
- MoldUDP protocols

### 6. Order Book Mechanics
- Price-time priority
- O(1) operations
- Depth aggregation

### 7. System Tuning
- isolcpus
- nohz_full
- TLB shootdowns
- Interrupt affinity

---

## 📊 Why These Matter for Top Firms

### Jane Street
Emphasizes functional programming (OCaml) but C++ for hot paths. Show you understand when to optimize.

### Citadel
Multi-strategy. Needs engineers who can work across different latency requirements.

### Jump Trading
Ultra-low latency. FPGA/hardware knowledge is a plus. Show kernel bypass experience.

### HRT (Hudson River Trading)
Quantitative focus. Combine systems skills with market microstructure models.

---

## 🎯 Final Recommendations

### Start with:
- CppTrader
- SPSCQueue
- Rigtorp's guide

### Build:
Your own order book engine using learned techniques

### Document:
Every optimization decision with benchmarks

### Study:
awesome-lockfree papers while building

### Specialize:
DPDK if targeting ultra-low latency roles

---

## 📋 Quick Reference Table

| Project | Performance | Difficulty | Time Investment | Portfolio Impact |
|---------|-------------|------------|-----------------|------------------|
| CppTrader | 41M msg/sec | High | 4-6 weeks | ⭐⭐⭐⭐⭐ |
| SPSCQueue | 362K ops/ms | Medium | 2-3 weeks | ⭐⭐⭐⭐⭐ |
| limit-order-book | 20.7M msg/sec | High | 4-6 weeks | ⭐⭐⭐⭐ |
| Anish's Article | 27M orders/sec | Expert | 3-4 weeks (study + implement) | ⭐⭐⭐⭐ |
| hft-infra-lab | 17.8M orders/sec | Expert | 8-10 weeks | ⭐⭐⭐⭐ |
| Liquibook | 2.5M inserts/sec | Medium | 3-4 weeks | ⭐⭐⭐ |
| concurrentqueue | Fastest MPMC | Medium-High | 2-3 weeks | ⭐⭐⭐⭐ |

---

## 📖 Additional Resources

### Books
- "Building Low Latency Applications with C++" - Sourav Ghosh
- "C++ High Performance" - Björn Andrist, Viktor Sehr
- "The Art of Multiprocessor Programming" - Maurice Herlihy, Nir Shavit

### Papers
- "Simple, Fast, and Practical Non-Blocking and Blocking Concurrent Queue Algorithms" - Michael & Scott
- "What Every Programmer Should Know About Memory" - Ulrich Drepper
- "A Primer on Memory Consistency and Cache Coherence" - Sorin, Hill, Wood

### Websites
- [1024cores.net](http://www.1024cores.net/) - Dmitry Vyukov's lock-free algorithms
- [mechanical-sympathy.blogspot.com](https://mechanical-sympathy.blogspot.com/) - Martin Thompson's blog
- [rigtorp.se](https://rigtorp.se/) - Erik Rigtorp's low-latency guides

---

**Created:** 2026-07-21  
**Purpose:** Portfolio guidance for HFT C++ engineer roles  
**Target Firms:** Jane Street, Citadel, Jump Trading, HRT, Optiver, IMC, Two Sigma
