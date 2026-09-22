# Module 14 — Exceptions, `noexcept`, error handling without exceptions

Every non-trivial function can fail — a division by zero, a malformed message, a full queue, a rejected
order. How you *signal* that failure and how the caller *handles* it is a design decision with real
latency consequences. General-purpose C++ leans on exceptions; HFT hot paths often ban them outright and
use status codes or `std::expected` instead. A strong candidate knows **both** mechanisms cold and can
articulate *why* each belongs where it does — that "I'd use exceptions here and codes there, for these
reasons" sentence is exactly what interviewers listen for.

---

## 1. Exceptions: the mechanism

An exception separates the *detection* of an error (deep in some function) from its *handling* (far up
the call stack), without every intermediate function having to check and forward a return code.

```cpp
double divide(double a, double b) {
    if (b == 0) throw std::runtime_error("divide by zero");
    return a / b;
}

try {
    auto r = divide(x, 0);
    use(r);
} catch (const std::exception& e) {   // catch by const reference — no slicing, no copy
    std::cerr << e.what();
}
```

Three things happen when `throw` fires:

1. An exception object is constructed (usually in a special exception-storage area, not the normal stack).
2. The stack **unwinds**: control walks *up* the call stack looking for a matching `catch`. As it leaves
   each frame, **every local object's destructor runs** — this is RAII (Module 7) doing cleanup. Locks
   release, files close, heap buffers free, all automatically.
3. When a matching `catch` is found, its body runs; execution continues after the `try/catch`.

**Always catch by `const&`.** Catching by value copies the exception and can **slice** a derived
exception down to its base (losing the real type and message). `catch (const std::exception&)` catches the
entire standard hierarchy (`runtime_error`, `logic_error`, `bad_alloc`, …) by reference.

### Memory picture — unwinding

```
call stack (grows down)          during throw: unwind upward, running dtors
┌───────────────────────┐
│ main()                │  ◀── catch found here → handler runs
│  try { … }            │
├───────────────────────┤
│ process()  locals L2  │  ── ~L2() runs as frame is discarded
├───────────────────────┤
│ divide()   locals L1  │  ── ~L1() runs, then throw propagates up
└───────────────────────┘
                              exception object lives in separate storage,
                              not in any frame being unwound.
```

Unwinding and RAII are a **matched pair**: exceptions are *safe* in well-written C++ precisely because
every resource is owned by an object whose destructor runs during unwinding. This is why "use RAII for
everything" and "exceptions are fine" are two sides of one coin.

---

## 2. Exception-safety guarantees

When you write a function, you're implicitly promising one of four levels of behavior on failure. Knowing
this vocabulary — and being able to classify a given function — is a standard interview probe.

- **No-throw guarantee**: the function never throws. Mark it `noexcept`. (Destructors, swaps, moves you
  intend to be fast should be here.)
- **Strong guarantee**: if it throws, program state is **completely unchanged** — commit-or-rollback,
  like a database transaction. The canonical technique is **copy-and-swap** (Module 7): do all the
  work that can throw on a *copy*, then swap it in with a `noexcept` swap.
- **Basic guarantee**: if it throws, no resources leak and all invariants still hold, but observable
  state *may* have changed (e.g. a container might have partially updated). This is the minimum a sane
  function should offer.
- **No guarantee**: throwing may leak or corrupt. Unacceptable — this is what RAII exists to eliminate.

Interview framing: "`push_back` on `std::vector` offers the strong guarantee — if the element's copy/move
throws during a reallocation, the vector is left unchanged — *provided* the element's move is `noexcept`,
otherwise it falls back to copying to preserve rollback." That single sentence ties three modules together
(9, 12, 14).

---

## 3. `noexcept` — the promise not to throw

```cpp
void f() noexcept;   // promise: never throws. If it does anyway -> std::terminate (hard crash)
```

`noexcept` is far more than documentation:

- **It enables optimizations.** The compiler needn't emit unwinding scaffolding around a `noexcept` call,
  and can keep more state in registers across it.
- **`std::vector` moves elements only if the element's move constructor is `noexcept`** (Module 9).
  Otherwise, to preserve the strong guarantee during a reallocation, it **copies** every element — turning
  an O(n) move into an O(n) deep copy. This alone makes `noexcept` on your move constructor effectively
  mandatory.
- **Destructors are implicitly `noexcept`.** *Never let a destructor throw.* If an exception is already
  propagating (mid-unwind) and a destructor throws a *second* exception, the runtime calls
  `std::terminate` — instant crash, no recovery.

A useful mental model: `noexcept` is a *narrow contract*. Break it and you don't get an exception — you
get `std::terminate`. So mark a function `noexcept` only when it genuinely cannot throw, and mean it.

---

## 4. The cost of exceptions (why HFT hot paths ban them)

Modern implementations (the Itanium C++ ABI, "zero-cost" / table-based exceptions used by GCC and Clang)
have a very specific cost profile that you must be able to state precisely:

```
Happy path (no throw):        cost ≈ 0
┌──────────────────────────────────────────┐
│ .text (hot):  normal code, no checks,     │  ← the try adds NO instructions on success
│               no branches for the throw   │
├──────────────────────────────────────────┤
│ .gcc_except_table / .eh_frame (cold):     │  ← lookup tables, only touched WHEN a throw happens
│   "if PC is in range X, run these dtors,  │
│    then jump to handler Y"                 │
└──────────────────────────────────────────┘

Throw path: walk the tables, unwind, run dtors → microseconds, UNBOUNDED & unpredictable
```

- **When no exception is thrown: truly ~zero runtime cost on the happy path.** This is real — a `try`
  block that doesn't throw runs exactly as fast as code without one. The overhead is moved into cold
  side-tables consulted only during an actual throw.
- **When an exception *is* thrown: expensive and *unbounded/unpredictable*** — table lookups, stack
  unwinding, destructor calls. This can take microseconds and the time is data-dependent.

So the HFT objection is **not** the happy-path cost (there is none). It's three other things:

1. **Unpredictable tail latency.** The one time an exception *does* fire, it blows the P99.9. HFT cares
   about the worst case, not the average — a rare microsecond-scale unwind on the matching path is
   unacceptable.
2. **Binary size / instruction-cache pressure.** Exception tables and generated cleanup code bloat the
   binary, evicting hot code from the i-cache (Module 15) and slowing the *happy* path indirectly.
3. **Determinism.** Some shops compile the whole hot path with `-fno-exceptions` for a provably
   exception-free, deterministic binary.

Conclusion: use exceptions for **exceptional, cold** conditions (startup misconfiguration, a malformed
message at the system boundary, an allocation failure) — **never for control flow on the hot path**.

---

## 5. Error handling without exceptions

On the matching path you want failure signaling that is explicit, predictable, and unwinding-free.

**1. `std::optional<T>` — "a value, or nothing":**
```cpp
std::optional<Order*> find(OrderId id);
if (auto o = find(id)) { (*o)->cancel(); }   // present
else { /* not found — a normal, expected outcome, not an error */ }
```
Use when absence is an ordinary, expected result (a lookup miss), not a failure with a reason.

**2. `std::expected<T, E>` (C++23) — "a value, or an error with a reason":**
```cpp
std::expected<Order, ErrorCode> parse(std::string_view msg);
auto r = parse(buf);
if (r) use(*r);
else handle(r.error());       // r.error() carries WHY it failed
```
This is the modern `Result` type: it forces the caller to confront both outcomes, carries the error
*reason*, and never unwinds. It's becoming the default for "can fail and I need to know why" on hot paths.

**3. Error codes / status enums — the classic, fully predictable path:**
```cpp
enum class Status { Ok, Rejected, Filled, Unknown };
Status addOrder(const Order& o) noexcept;   // no throw; caller inspects the return
```
Zero surprise, zero unwinding, trivially predictable latency — the workhorse of hot-path APIs.

**4. `std::error_code` — for system/library errors** you want to report without throwing.

The trade-off vs exceptions: codes/`optional`/`expected` are **predictable and unwinding-free** but
**verbose** and easy to ignore (a discarded return value silently drops an error — `[[nodiscard]]` helps).
Exceptions give a **clean happy path** and can't be silently ignored, at the price of unpredictable throw
cost and binary bloat.

---

## 6. Common pitfalls & UB

- **Throwing from a destructor** (especially during unwinding) → `std::terminate`. Destructors are
  `noexcept` by default; keep them that way.
- **Throwing across a `noexcept` boundary** → `std::terminate`, immediately, with no chance to catch.
- **Catch by value** → slicing (a `bad_alloc` caught as `std::exception` by value loses its type) and an
  extra copy. Catch by `const&`.
- **Using exceptions for control flow** (e.g. "throw to break out of a loop") — abuses the unbounded slow
  path and is a design smell.
- **Ignoring a returned status/`expected`** silently swallows the error. Mark such functions
  `[[nodiscard]]`.
- **`throw;` vs `throw e;` in a handler**: bare `throw;` re-throws the *current* exception preserving its
  dynamic type; `throw e;` copies (and may slice). Use bare `throw;` to rethrow.

---

## 7. Quiz — tough problems

**Q1.** What does this print?
```cpp
struct Guard { ~Guard() { std::cout << "G"; } };
void g() { Guard x; throw std::runtime_error("boom"); }
int main() {
    try { g(); } catch (const std::exception&) { std::cout << "C"; }
}
```
**Answer:** `GC`. Throwing in `g()` unwinds its frame, running `~Guard` (prints `G`) *before* control
reaches the handler (prints `C`). RAII cleanup happens during unwinding, before the catch body.

**Q2.** Why might this program `std::terminate` instead of printing "caught"?
```cpp
struct Bad { ~Bad() { throw 1; } };
int main() {
    try { Bad b; throw std::runtime_error("x"); }
    catch (...) { std::cout << "caught"; }
}
```
**Answer:** The `throw` starts unwinding; unwinding destroys `b`, whose destructor throws a *second*
exception while the first is in flight. Two simultaneous exceptions → `std::terminate`. Destructors must
not throw.

**Q3.** Classify the exception-safety guarantee:
```cpp
void update(std::vector<int>& v, int x) {
    v.push_back(x);        // (A)
    v.push_back(x * 2);    // (B)
}
```
**Answer:** **Basic**, not strong. If (B) throws (e.g. on reallocation with a throwing element type — not
`int`, but in general), (A) has already mutated `v`. State changed but no leak/corruption → basic. To make
it strong you'd build both into a temporary and swap.

**Q4.** True or false: adding a `try/catch` around a hot loop that never throws slows the loop down.
**Answer:** Essentially false with table-based (zero-cost) exceptions — the happy path emits no extra
instructions; the cost lives in cold side-tables consulted only on an actual throw. (Indirect i-cache/
binary-size effects aside.)

**Q5.** Why does removing `noexcept` from this move constructor hurt `std::vector<Buffer>` performance?
```cpp
Buffer(Buffer&& o) /* noexcept? */ { steal(o); }
```
**Answer:** During reallocation `std::vector` needs the strong guarantee. It will *move* elements only if
the move is `noexcept`; without `noexcept` it must **copy** them (so it can roll back if a copy throws).
So a missing `noexcept` silently turns every growth into a full deep copy — O(n) copies instead of O(n)
cheap moves.

**Q6.** `std::optional<int>` vs `std::expected<int, Error>` — when do you pick each?
**Answer:** `optional` when absence is a normal, reasonless outcome (a cache miss). `expected` when a
failure has a *reason* the caller must see (parse failed *because* the length field was invalid). `expected`
carries the error payload; `optional` only says yes/no.

**Q7.** What's the difference between `throw;` and `throw e;` inside a `catch (std::exception& e)`?
**Answer:** `throw;` **rethrows the current exception object** preserving its dynamic (most-derived) type.
`throw e;` throws a *copy* of `e` typed as `std::exception` → slicing, losing the derived type. Use bare
`throw;` to propagate.

**Q8.** On the matching hot path, `addLimitOrder` can be rejected (risk limit) or filled. Do you throw on
rejection? **Answer:** No — rejection is a *normal, expected* outcome, not an exceptional one, and the hot
path must be deterministic. Return a `Status`/`enum` (or `expected`) and mark the function `noexcept`.
Reserve `throw` for cold, truly-exceptional conditions off the hot path.

---

## 8. Indian HFT interview questions

Common at Tower Research (Gurgaon), Optiver, IMC, Graviton, Quadeye, AlphaGrep, WorldQuant, Da Vinci,
Squarepoint, Jump, Hudson River, NK Securities, Mansard, and Qube.

**"What's the runtime cost of a C++ exception?"**
With modern table-based ("zero-cost") implementations: ~zero on the happy path — a `try` that doesn't
throw runs as fast as code without one, because the overhead lives in cold side-tables. But *throwing* is
expensive and **unbounded/unpredictable** (table walk + unwind + destructors, microseconds, data-
dependent). Average-cheap, worst-case-terrible — the opposite of what a P99-latency system wants.

**"Why do HFT firms ban exceptions on the hot path if they're zero-cost when not thrown?"**
Three reasons: (1) the rare throw destroys tail latency (P99.9) and the time is unbounded/unpredictable;
(2) exception tables and cleanup code bloat the binary and pressure the instruction cache, indirectly
slowing the happy path; (3) determinism — some shops build with `-fno-exceptions`. Failure signaling on
the hot path is done with status enums / `std::expected` instead.

**"Name the exception-safety guarantees."**
No-throw (`noexcept`), strong (commit-or-rollback, e.g. copy-and-swap), basic (no leak, invariants hold,
state may change), and none (avoid). Be ready to classify a given function and to explain copy-and-swap as
the strong-guarantee technique.

**"Why must a move constructor be `noexcept`?"**
Because `std::vector` (and similar) will only *move* elements during reallocation if the move is
`noexcept` — otherwise it *copies* to preserve the strong guarantee on rollback. A missing `noexcept`
silently degrades every container growth from cheap moves to full deep copies.

**"Can a destructor throw?"**
It shouldn't. Destructors are implicitly `noexcept`; throwing from one (especially during unwinding, when
another exception is already live) calls `std::terminate`. Destructors must be no-throw.

**"How do you handle errors on the hot path without exceptions?"**
Status enums / error codes for expected outcomes, `std::expected<T,E>` when you need the failure *reason*,
`std::optional<T>` for reasonless absence, `std::error_code` for system errors. All predictable, unwinding-
free, `noexcept`. Mark them `[[nodiscard]]` so a caller can't silently drop the error.

**"exceptions vs error codes — how would you decide in a real system?"**
Split by hot vs cold. Cold/exceptional (startup config error, malformed boundary input, `bad_alloc`):
exceptions — clean, can't be ignored, cost only when they fire. Hot/expected (order rejected, queue full,
lookup miss): status codes / `expected` — deterministic latency, no unwinding. State it as one sentence:
"exceptions for exceptional conditions off the hot path, codes/`expected` on the matching path for
deterministic latency."

---

## 9. In the order book

- **Matching path: no exceptions.** `addLimitOrder` returns a `Status`/`enum` (`Ok`, `Rejected`,
  `Filled`), and every hot-path function is `noexcept`. Rejection and fills are *normal* outcomes, not
  exceptional ones, so they must be expressed as return values with deterministic cost — not thrown.
- **Cold path: exceptions are fine and clearer.** Parsing a malformed inbound message at the boundary, a
  bad config file at startup, an allocation failure while building the pool — these are genuinely
  exceptional, off the hot path, and reading cleaner with `throw` + a single boundary `catch`.
- `std::optional<Order*>` (or an index sentinel) for "find order by ID," since a miss is an ordinary
  expected result. `std::expected` for parse routines that need to report *why* a message was rejected.
- This exceptions-cold / codes-hot split is precisely the design you'd defend in an interview, and it
  ties directly back to `noexcept` moves (Module 9) enabling cheap `std::vector<Trade>` growth for the
  fills log.

---

## Key takeaways

- Exceptions separate error *detection* from *handling* and unwind the stack, running every local's
  destructor — they're safe **because** RAII cleans up during unwinding. Catch by `const&`.
- Know the four **exception-safety guarantees** (no-throw, strong, basic, none) and copy-and-swap as the
  strong-guarantee technique.
- `noexcept` is a hard promise (violate → `std::terminate`); it enables optimizations and is what lets
  `std::vector` *move* rather than *copy* on reallocation. Destructors are `noexcept` — never throw from
  one.
- Exceptions are **~zero-cost on the happy path but unbounded/unpredictable when thrown**, and they bloat
  the binary — so HFT hot paths ban them for tail-latency and determinism reasons.
- On the hot path, signal failure with **status enums, `std::expected`, `std::optional`, `error_code`** —
  predictable, unwinding-free. Reserve exceptions for cold, truly-exceptional conditions.

**Next:** [15 — Memory model, cache, alignment, false sharing](15-memory-cache.md) — how the hardware
actually decides your latency.
