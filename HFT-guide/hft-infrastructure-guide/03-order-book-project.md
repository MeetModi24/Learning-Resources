# Module 03 — Order-book architecture: your project, dissected & optimized

This module is a **guide to the order book you actually built**
([github.com/MeetModi24/Limit-Order-Book](https://github.com/MeetModi24/Limit-Order-Book)) — not an
idealized textbook version. It does three things:

1. **Documents what you built** — the real classes, data structures, and matching logic, with the
   concrete complexity of each operation.
2. **Names every concept in play** — the C++, computer-architecture (CA), and OS ideas your code
   already exercises, so you can defend each one in an interview.
3. **Lays out the rung-1 → HFT optimization path** — specific, code-level changes (not generic
   advice), each tied to the exact line/structure it fixes and the latency it buys back.

Your current implementation is a real, defensible **rung-1**: AVL trees of price levels, intrusive
FIFO linked lists per level, hash maps for lookup, full stop / stop-limit support, ~1.4M TPS at
~713 ns average on an i5-12450H. This module treats that as the baseline and shows how an HFT
engineer would push it to **rung-3**.

> Naming convention: **rung 1** = correct & reasonably fast (what you have). **rung 3** =
> cache-optimized, allocation-free, measured (the differentiation). We'll refer back to C++ modules
> (cpp-guide 1–18) by number.

---

## 1. What a limit order book does (the semantics you implemented)

An exchange matches buyers and sellers. The **order book** holds every resting order and decides what
trades. Resting **bids** (buys) and **asks** (sells) are grouped by price. On a new order:

- If it **crosses** — a buy priced ≥ best ask, or a sell priced ≤ best bid — it **matches**: trades
  execute immediately against resting orders.
- Any leftover quantity **rests** at its price, waiting for a counterparty.

The rule is **price–time priority (FIFO)**: best price served first; within a price, earliest arrival
filled first. Your engine implements exactly this — and adds **stop** and **stop-limit** orders on
top.

```
        ASKS (sell)                 BIDS (buy)
price   qty   orders                price  qty  orders
102     50    [o7]                  100    30   [o1,o4]   <- best bid (highestBuy)
101     20    [o5,o6]  <- best ask  99     80   [o2]      (lowestSell)
                                    98     10   [o3]
                        ^ spread = bestAsk - bestBid = 101 - 100 = 1
```

A buy for 25 @ 101 crosses the best ask: fills 20 vs `o5`, then 5 vs `o6` (FIFO within the level),
no leftover. A buy for 25 @ 100 doesn't cross, so it rests at 100 behind `o1`,`o4`. In your code the
"does it cross?" step is `limitOrderAsMarketOrder(...)`, and the "match against the edge" step is
`marketOrderHelper(...)` walking `lowestSell` / `highestBuy`.

**Order types you support** (from `Book.hpp`): market, limit (add/modify/cancel), market-limit
(a limit that crosses and trades immediately), stop (add/modify/cancel), stop-limit
(add/modify/cancel). That stop/stop-limit machinery — a second pair of AVL trees fired by
`executeStopOrders()` — is genuinely more than most student order books, and worth leading with in
interviews.

---

## 2. Your architecture, as built

Four cooperating structure *families*, all in `Book`:

```
PRICE LEVELS: two AVL trees keyed by price          ORDERS WITHIN A LEVEL: intrusive doubly-linked FIFO
(buyTree, sellTree) + edge ptrs                     (headOrder … tailOrder)

        sellTree (asks)                              Limit(price=101):
             (102)                                     headOrder → [o5] ⇄ [o6] ← tailOrder
            /     \                                                (oldest)   (newest)
       (101)      (103)   ◀── lowestSell             match from head, append at tail = time priority
        ⇧ each node is a Limit; each Limit
          owns a DLL of Orders

  HASH MAPS (unordered_map)                          STOP LEVELS: a second pair of AVL trees
  orderMap:     id     → Order*                       stopBuyTree / stopSellTree keyed by stopPrice
  limitBuyMap:  price  → Limit*                       + stopMap: stopPrice → Limit*
  limitSellMap: price  → Limit*                       fired when a trade crosses a stop price
  stopMap:      price  → Limit*                       via executeStopOrders(side)
```

### 2.1 `Order` (a node in a price level's FIFO list)

```cpp
class Order {
    int    idNumber;      // unique id
    bool   buyOrSell;     // true = buy
    int    shares;        // remaining quantity
    int    limit;         // price (integer)
    Order *nextOrder;     // intrusive DLL links
    Order *prevOrder;
    Limit *parentLimit;   // back-pointer to its price level
    friend class Limit;
    // + partiallyFillOrder / cancel / execute / modifyOrder
};
```

`cancel()` and `execute()` splice the node out of its list in **O(1)** using `prev`/`next` and adjust
the parent `Limit`'s `size`/`totalVolume` — this is why cancel is O(1) (see §3).

### 2.2 `Limit` (a price level = an AVL tree node)

```cpp
class Limit {
    int    limitPrice;
    int    size;          // # orders at this price
    int    totalVolume;   // sum of shares at this price
    bool   buyOrSell;
    Limit *parent, *leftChild, *rightChild;   // AVL tree pointers
    Order *headOrder, *tailOrder;             // FIFO list of orders
    // NOTE: no cached `height` field  ← this is a real inefficiency (see §7.2)
};
```

`append(Order*)` pushes an order at the tail (time priority) and updates `size`/`totalVolume` in O(1).

### 2.3 `Book` (the engine)

```cpp
class Book {
    Limit *buyTree, *sellTree;              // AVL trees of price levels
    Limit *lowestSell, *highestBuy;         // O(1) best-ask / best-bid pointers
    Limit *stopBuyTree, *stopSellTree;      // AVL trees of stop levels
    Limit *highestStopSell, *lowestStopBuy;
    std::unordered_map<int, Order*> orderMap;        // id    → order   (O(1) cancel lookup)
    std::unordered_map<int, Limit*> limitBuyMap;     // price → level
    std::unordered_map<int, Limit*> limitSellMap;
    std::unordered_map<int, Limit*> stopMap;
    // + AVL rotate/balance, insert, edge maintenance, matching helpers …
};
```

The **edge pointers** (`lowestSell`/`highestBuy`) are the trick that makes best-bid/ask **O(1)**: you
don't walk the tree to the leftmost node on every order; you keep a direct pointer and update it in
O(1) on insert (`updateBookEdgeInsert`) and delete (`updateBookEdgeRemove`). Maintaining those
correctly is why each `Limit` needs a `parent` pointer.

---

## 3. The operations & their real complexity

| Operation | Path in your code | Complexity | Notes |
|---|---|---|---|
| **Add limit** (rests) | `addLimitOrder` → `limitOrderAsMarketOrder` (match) → `addLimit`+`append` | O(log M) first order at a new price (AVL insert), **O(1)** for subsequent orders at an existing price | M = # active price levels (~10k) |
| **Cancel** | `cancelLimitOrder` → `orderMap` lookup + `order->cancel()` splice; `deleteLimit` if level empties | **O(1)** splice; O(log M) *if* the level empties (AVL delete + rebalance) | doubly-linked list is what makes the splice O(1) |
| **Modify** | `modifyLimitOrder`: cancel + re-append at new price | O(1), or O(log M) if a level appears/empties | correctly loses time priority (goes to tail) |
| **Market** | `marketOrder` → `marketOrderHelper` walks `lowestSell`/`highestBuy` | O(1) per fill | then `executeStopOrders` |
| **Best bid/ask** | read `highestBuy` / `lowestSell` | **O(1)** | edge pointers, no tree walk |
| **Stop trigger** | `executeStopOrders(side)` after each trade | O(triggered stops) | second AVL pair |

The matching loop in `marketOrderHelper` is the heart of price–time priority:

```cpp
while (bookEdge != nullptr && bookEdge->getHeadOrder()->getShares() <= shares) {
    Order* head = bookEdge->getHeadOrder();
    shares -= head->getShares();
    head->execute();                       // splice out of FIFO list (O(1))
    if (bookEdge->getSize() == 0) deleteLimit(bookEdge);  // level emptied → AVL delete
    deleteFromOrderMap(head->getOrderId());
    delete head;                           // ← heap free on the hot path (see §7.1)
    executedOrdersCount++;
}
if (bookEdge != nullptr && shares != 0)
    bookEdge->getHeadOrder()->partiallyFillOrder(shares);   // partial fill of the next order
```

`bookEdge` (the map's-not-needed edge pointer) gives price priority; taking `headOrder` first and
appending new orders at the tail gives time priority. This is correct — the FIFO discipline is
exactly right.

---

## 4. The harness around the engine

Your project isn't just the book — it's a full test/benchmark rig, which is a big part of what makes
it interview-worthy:

- **`GenerateOrders`** — a statistical order generator: prices ~ Normal(μ=300, σ=50), ~10k active
  limits and ~1k active stops on average, 5M requests over a run. This is *how you get realistic data*
  — a question interviewers ask ("where did your test data come from?").
- **`OrderPipeline`** — reads a text file line by line and dispatches each to the right handler via
  `std::unordered_map<std::string_view, void(OrderPipeline::*)(std::istringstream&)>` (a
  **member-function-pointer** table keyed by the order-type token). Clean design; but note it hashes a
  string and constructs an `istringstream` per line — and that cost is *inside your timed region*
  (see §6).
- **`main.cpp`** — times `processOrdersFromFile("Orders.txt")` with
  `std::chrono::high_resolution_clock`.
- **Build/test** — CMake (C++20, `-O2`), GoogleTest unit + integration tests, Python
  (`data_visualisation.py`) for the latency histograms.

---

## 5. Concepts your project already exercises

### 5.1 C++ (cpp-guide modules)

| Concept | Where in your code | Module |
|---|---|---|
| Classes, encapsulation, `friend` | `Order`/`Limit`/`Book`; `friend class Limit` | 6 |
| Constructors/destructors | `Book()`/`~Book()`, `Limit()`/`~Limit()` | 6 |
| Pointers & references | intrusive `next/prev/parent`, `auto& tree = buyOrSell ? buyTree : sellTree` | 2 |
| Dynamic memory (`new`/`delete`) | `new Order`, `new Limit`, `delete` in matching | 3 |
| **RAII (mostly *absent*)** | you manage memory **manually** with raw `new`/`delete` — not RAII | 7, 13 |
| STL containers | `unordered_map`, `unordered_set`, `vector` | 12 |
| `std::string_view` | pipeline dispatch keys (no copy of the token) | 18 |
| Member function pointers | `OrderFunction` dispatch table | 6, 10 |
| `<random>` | `std::mt19937`, `normal_distribution` in the generator | 12 |
| `<chrono>` | latency timing in `main` | 12 |
| Templates | via GoogleTest; not in your own hot path yet | 10 |
| C++20 standard set | `set(CMAKE_CXX_STANDARD 20)` — but `<bit>`, concepts, `span` **unused** | 18 |

The honest read: your code is solid **classical OOP C++**. The *systems* C++ from cpp-guide Part II
(cache layout, atomics, zero-cost abstraction, `<bit>`) is exactly what rung-3 adds.

### 5.2 Computer architecture (where it bites *this* code)

- **Pointer chasing → cache misses.** An AVL node (`Limit`) and each `Order` are separate heap
  allocations scattered in memory. Walking the tree, or hopping `head→next→next…`, is a chain of
  dependent loads, each a potential cache miss (~tens–hundreds of ns). This is the dominant hidden
  cost, and it's *structural* — a direct consequence of pointers + per-object `new`.
- **Branch prediction.** `bool buyOrSell` fans out into `? :` and `if` on nearly every operation. The
  branch predictor handles it well when order flow is one-sided, worse when it alternates.
- **AVL rebalancing** (your README's measured bottleneck: ~2500 ns per rebalance, 252k rebalances over
  5M orders) — rotations touch several cold nodes and, in your implementation, recompute subtree
  heights (see §7.2).
- **Data locality** — nothing in your layout is contiguous, so the hardware prefetcher can't help you.

### 5.3 OS (what you use, and what HFT would add)

- **Currently used:** file I/O (`ifstream`/`ofstream`), the process heap allocator (every `new`), a
  monotonic clock. Single-threaded, one process. That's it — and that's fine for rung 1.
- **What rung-3/HFT adds (none of it present yet, all fair game to discuss):** CPU **affinity /
  core pinning** (`taskset`, `sched_setaffinity`) so the matching thread never migrates; **isolated
  cores** (`isolcpus`) and **busy-polling**; **huge pages** to cut TLB misses on the big pool;
  `mmap`-ing the input file instead of `istringstream`; `perf stat` / `perf record` to read cache-miss
  and branch-miss counters. Knowing these exist and *why* is the OS half of an HFT interview.

---

## 6. Your measured performance — and reading it honestly

From your README (i5-12450H, 5M orders): **~713 ns average, ~1.4M TPS**; cancel ~400 ns; add/modify
~700 ns; **each AVL rebalance ~+2500 ns**; 845k trades; 252k rebalances. Your own analysis correctly
fingers **AVL rebalances** as the biggest latency driver.

Two honest caveats to raise *before* an interviewer does:

1. **Average hides the tail.** You have a histogram — always quote **P50 / P99 / P99.9**, never just
   the mean. HFT cares about the worst 0.1%, which is set by cache misses, rebalances, and any
   allocation.
2. **Your timed region includes parsing.** `main` times `processOrdersFromFile`, which parses text
   (`istringstream` + string-keyed dispatch) *and* matches. So 713 ns is *parse + match*, not
   match-only. Separating the two (or pre-parsing to a binary buffer) will both lower the number and
   make it honest about what the *engine* costs.

---

## 7. The rung-1 → rung-3 optimization path (specific to your code)

Ordered by impact-per-effort. Each item names the exact thing it fixes and the concept it exercises.

### 7.1 Kill `new`/`delete` on the hot path → object pool *(biggest determinism win; Modules 3, 7, 15)*

**Problem:** `addLimitOrder` does `new Order`, `marketOrderHelper`/`cancel` do `delete`. Every order
touches the general allocator — hundreds of ns, non-deterministic, can take a lock or a syscall,
scatters objects across memory (killing locality). This is *the* classic HFT anti-pattern and it's on
your hottest path.

**Fix:** pre-allocate a pool of `Order` (and `Limit`) once at startup; `acquire()`/`release()` from a
free-list.

```cpp
class OrderPool {
    std::vector<Order> storage_;        // allocated ONCE
    std::vector<uint32_t> free_;        // free slot indices
public:
    explicit OrderPool(size_t n) : storage_(n) {
        free_.reserve(n);
        for (uint32_t i = 0; i < n; ++i) free_.push_back(n - 1 - i);
    }
    uint32_t acquire() { auto i = free_.back(); free_.pop_back(); return i; } // O(1)
    void release(uint32_t i) { free_.push_back(i); }                          // O(1)
    Order& operator[](uint32_t i) { return storage_[i]; }
};
```

**Impact:** removes an unpredictable heap op from every add and every fill; tightens the tail
dramatically; also gives contiguous storage (better locality). **Effort: low. Do this first.**

### 7.2 Cache the AVL height (or drop the AVL entirely) *(Modules 12, 15, 18)*

**Problem I found:** `Limit` has **no `height` field**, so `getLimitHeight` recurses the *entire*
subtree, and `balance()` (which calls it via `limitHeightDifference`) runs at *every* node on the
insert/delete path. So each insert is far worse than O(log M) in constants — it's roughly O(M) work
per structural change. That inflates your 2500 ns rebalance cost.

**Two options:**

- **Cheap fix (keep the AVL):** add `int height;` to `Limit`, update it in O(1) during rotations.
  Now `balance` is genuinely O(log M). This is a small change with a large, measurable win — a great
  *before/after* data point on its own.
- **The differentiation (replace the AVL):** swap the price-level tree for a **flat array indexed by
  price tick + a bitset of occupied levels**, and find best-bid/ask with `std::countr_zero`
  (`<bit>`, Module 18) — one instruction, no rebalancing, cache-friendly contiguous levels.

```cpp
static constexpr size_t Levels = 1 << 16;
struct Level { uint32_t head, tail, totalVol; };
std::array<Level, Levels> levels_;                 // price tick → level (contiguous)
std::array<uint64_t, Levels/64> occupied_{};       // 1 bit per level

size_t bestAskLevel() const {                      // lowest occupied level
    for (size_t w = 0; w < occupied_.size(); ++w)
        if (occupied_[w]) return w*64 + std::countr_zero(occupied_[w]);
    return Levels;                                 // empty
}
```

**Impact:** *eliminates the rebalance bottleneck your own README identified.* This is the headline
rung-3 result. **Tradeoff to state honestly:** the array costs memory ∝ price *range*, not order
count — ideal for a bounded tick band (equities/futures), wasteful for a huge sparse range; mitigate
with a windowed array around the touch, or a hybrid array-near-touch + map-for-tails. Naming that
tradeoff is exactly the judgment interviewers reward.

### 7.3 `unordered_map` → direct-indexed array or flat hash *(Modules 12, 15)*

**Problem:** `orderMap`/`limitMaps`/`stopMap` are node-based `std::unordered_map` — every insert
allocates a node, lookups chase pointers into cold cache lines, and it's a per-order cost.

**Fix:** your generated order IDs are **sequential from 11001**, so `orderMap` can be a plain
`std::vector<uint32_t>` indexed by `(id - base)` → pool slot: O(1), zero allocation, perfect locality.
For prices, an open-addressing **flat hash map** (e.g. `absl::flat_hash_map` / `ankerl::unordered_dense`)
or, better, fold price lookup into the array-of-levels from §7.2 (the index *is* the price). **Impact:**
removes hash + pointer-chase + node allocation from every add/cancel.

### 7.4 Pointers → 32-bit indices, pack for cache lines *(Module 15)*

**Problem:** `Order` and `Limit` link with 64-bit pointers; objects are scattered. **Fix:** once
objects live in pools (§7.1/§7.2), replace `Order*`/`Limit*` links with 32-bit **indices** into the
pool — half the size, survive pool relocation, and keep hot fields packed within a cache line. Order
the struct so the fields touched on the hot path share one 64-byte line. **Impact:** more orders per
cache line → fewer misses → lower tail.

### 7.5 Remove the `bool buyOrSell` branching *(Module 17)*

**Problem:** side is decided at runtime on nearly every call (`? :`, `if`). **Fix:** template the
side-specific paths (`Book::match<Side::Buy>()`), so the compiler specializes and drops the branch;
or keep symmetric buy/sell arrays indexed by `side` so it's a branchless index. **Impact:** modest,
but real on alternating flow, and it's a clean way to show you know zero-cost abstraction.

### 7.6 Faster parsing / honest measurement *(Modules 15, 18)*

**Problem:** `istringstream` + string-keyed dispatch per line, inside the timed region. **Fix:**
pre-parse the file into a packed binary array of order structs once, then time *only* the matching
loop over that array; or hand-roll integer parsing with `std::from_chars` (no allocation). **Impact:**
lower and *honest* engine-latency numbers; separates parser cost from matcher cost.

### 7.7 Decouple I/O with a lock-free SPSC queue; keep matching single-threaded; shard by symbol *(Module 16)*

Keep the matching core **single-threaded** — sharing one book across threads adds locking/coherency
cost and correctness risk that usually outweigh the gain. Decouple the parser/producer from the
matcher/consumer with a lock-free **SPSC ring** (`alignas(64)` head/tail to avoid false sharing,
Module 16). To use all cores, **shard by symbol**: one single-threaded engine per instrument, each
pinned to a core. (Your project is one book/one symbol today — this is the scaling story to *describe*.)

---

## 8. Complexity: current vs optimized

| Operation | Rung 1 (yours: AVL + unordered_map + new/delete) | Rung 3 (array+bitset + pool + index map) |
|---|---|---|
| Add resting order | O(log M) at new price *(worse in practice: uncached height)* + heap alloc | **O(1)** index + bit-set, pool acquire |
| Cancel | O(1) splice; O(log M)+heap free if level empties | **O(1)** splice + pool release |
| Best bid/ask | O(1) edge pointer | O(1) `countr_zero` (branch-free) |
| Match one fill | O(1) + heap free | **O(1)**, no alloc |
| Rebalance cost | ~2500 ns each (measured) | **none** |
| Allocation on hot path | yes (`new`/`delete` per order) | **none** |
| Memory | O(active orders) | O(price range), fixed |

Note the asymptotics barely change — the win is **constants and cache behavior**, which is *exactly*
why you must **measure**, not argue from big-O.

---

## 9. Pitfalls & gotchas — some specific to your code

- **Uncached AVL height (§7.2)** — the single most impactful correctness-of-*claims* issue: your add
  isn't really O(log M). Fix or acknowledge it.
- **Raw `new`/`delete`, no RAII (Module 7).** Manual lifetime is leak-prone under any early return or
  exception, and slow. Pool it. (You don't throw today, so no leak now — but it's fragile.)
- **`getRandomOrder(int key, std::mt19937 gen)` takes the RNG *by value*** — copies ~5 KB of engine
  state per call. Pass by reference. (Generator path, not the hot path, but a real gotcha.)
- **Timed region includes parsing (§6)** — separate parse from match before quoting engine latency.
- **Reporting the average** — always P50/P99/P99.9.
- **Deep recursion** in `insert`/tree traversals — fine at M≈10k, but iterative versions avoid stack
  growth and are friendlier to the optimizer.
- **`double` for price** — you correctly use `int`, keep it (integer ticks are exact; doubles have
  rounding + NaN-ordering problems).

---

## 10. Interview drills — framed around *your* project

**Q. "Walk me through your order book."**
Lead with semantics (price–time priority, the order types incl. stop/stop-limit), then the structures
(AVL trees of `Limit` price levels + edge pointers for O(1) best-bid/ask, intrusive doubly-linked FIFO
list per level for O(1) cancel and time priority, `unordered_map` for O(1) id lookup), then the
numbers (1.4M TPS, 713 ns avg, and — crucially — that *you profiled it* and found AVL rebalancing at
~2500 ns dominates), then how you'd push to rung-3. That arc — build → measure → find the bottleneck →
know the fix — is the whole interview.

**Q. "Your README says add is O(log M). Is it, really?"**
Be honest: the AVL insert is O(log M) *nodes on the path*, but because `Limit` doesn't cache its
height, `balance` recomputes subtree heights recursively, so each insert does ~O(M) extra work.
Caching the height fixes it; replacing the tree with an array+bitset removes rebalancing entirely.
(Volunteering this is a huge credibility signal.)

**Q. "What's the biggest latency cost and how would you remove it?"**
Two: (1) AVL rebalancing (my measured bottleneck) → array indexed by price tick + bitset, best-price
via `countr_zero`, no rebalance; (2) `new`/`delete` per order → object pool. Both attack determinism
(the tail), which is what matters.

**Q. "How is cancel O(1)?"**
`orderMap` finds the order in O(1); the per-level list is **doubly** linked, so I splice it out via
`prev`/`next` in O(1) — no scan. Both structures are required; either alone forces a search.

**Q. "How did you get test data, and how did you measure?"**
A statistical generator (prices Normal(300,50), ~10k active limits) producing 5M requests; per-order
timestamps around the matching call; a latency histogram analyzed in Python. I'd tighten it by timing
match-only (excluding parse) and reporting P99/P99.9.

**Q. "Single- or multi-threaded matching?"**
Single-threaded per book — avoids locking/coherency cost and is usually faster. Scale by sharding on
symbol, one pinned engine per instrument, fed by a lock-free SPSC queue.

**Q. "Why not `std::map` for price levels?"**
`std::map`/AVL rebalances and chases cold pointers. For a bounded tick range I'd use a price-indexed
array + bitset — O(1) add, one-instruction best-price, no rebalance — keeping the tree only as a
correctness reference, and I'm explicit about the memory-∝-range tradeoff.

---

## 11. Build order for the rung-3 upgrade

1. **Object pool** for `Order` (and `Limit`) — remove all hot-path `new`/`delete`. Re-benchmark.
2. **Cache the AVL `height`** — cheap, isolates the rebalance win. Re-benchmark.
3. **Array-of-levels + bitset** replacing the AVL — the headline change. Keep the AVL build as the
   differential-testing oracle. Re-benchmark.
4. **Direct-indexed `orderMap`** (IDs are sequential) / flat hash. Re-benchmark.
5. **Separate parse from match**; report P50/P99/P99.9 for each stage.
6. **(Optional)** SPSC queue + pinned matcher thread; measure tail under load.
7. **Rewrite the README** with the before/after table, the flame graph, and the *why* — that document
   is your interview.

Do them one at a time and **measure after each** — a per-change delta table ("pool: −X ns tail;
array+bitset: −Y ns; ...") is far more convincing than one big rewrite.

---

## Key takeaways

- Your project is a legitimate **rung-1**: AVL price-level trees + intrusive FIFO lists + hash maps,
  with full stop/stop-limit support — correct, tested, ~1.4M TPS.
- It already exercises real **C++** (OOP, STL, pointers, `string_view`, member-fn dispatch, `<random>`,
  `<chrono>`) and surfaces real **CA** costs (pointer-chasing cache misses, AVL rebalancing,
  branching). The **systems** C++ and **OS** tuning are what rung-3 adds.
- The two highest-impact upgrades are **an object pool** (kills hot-path allocation → tail latency)
  and **replacing the AVL with an array+bitset** (kills the rebalance bottleneck *you measured*).
- A found bug-in-claims: **`Limit` caches no height**, so inserts do ~O(M) work, not O(log M) — fix or
  disclose it.
- Always report **P50/P99/P99.9**, and **time match-only** (parsing is currently inside your timed
  region).
- The portfolio value is the **build → measure → diagnose → optimize** arc, told with per-change
  before/after numbers.

**Next:** Module 04 — [the strategy engine](04-strategy-engine.md): the brain that reads this book and
decides what to trade (OMS, recorder, backtester, and simulator are planned after it). Return to the
[index](../00-index.md), or start the rung-3 upgrade — object pool first.
