# Module 7 — RAII & the Rule of 0/3/5

The single most important idiom in C++. Everything about safe resource handling flows from it. This
module builds it up from the problem it solves, not from rules to memorize — because if you
understand *why* the compiler's defaults betray you the moment you own a raw resource, the rules
become obvious instead of arbitrary. This is also the module HFT interviewers probe hardest: it sits
at the intersection of correctness (no leaks, no double-frees) and performance (`noexcept` moves,
deterministic destruction, no GC pauses).

---

## 1. The problem: cleanup you have to remember is cleanup you'll forget

A **resource** is anything you *acquire and must later release*: heap memory (`new`→`delete`), a file
(`fopen`→`fclose`), a lock (`lock`→`unlock`), a socket, a database handle. The naive approach is to
release it by hand:

```cpp
void f() {
    int* data = new int[100];   // acquire
    doWork(data);
    delete[] data;              // release — IF we get here
}
```

This leaks on **every path that skips the `delete`**:

- an early `return` above it,
- a thrown **exception** from `doWork` (control jumps straight out of the function),
- someone editing the function six months later and adding a `return` without noticing the cleanup
  below it.

Manual release is fragile *because* it depends on a human remembering it on **every** exit path, and
a function can have many exit paths — some of which (exceptions) aren't even visible in the source.
RAII removes the human from the loop entirely.

Think of it like a rented apartment where the lease automatically ends and the keys automatically
return to the landlord the instant you step out the door — no matter which door, no matter whether
you left calmly or the building caught fire. You physically cannot forget to return the keys.

---

## 2. RAII: Resource Acquisition Is Initialization

The idea in one line: **tie the resource's lifetime to an object's lifetime.** Acquire the resource
in a **constructor**; release it in the **destructor**. Then lean on the one guarantee C++ gives you
for free (Modules 1 and 6): *a stack object's destructor always runs when it leaves scope — including
while an exception is unwinding the stack.*

So if the release lives in a destructor, it **cannot be skipped**. You're no longer relying on memory
or discipline; you're relying on the language's scope-exit machinery, which is as reliable as the `}`
itself.

```cpp
class Lock {
    std::mutex& m_;
public:
    explicit Lock(std::mutex& m) : m_{m} { m_.lock(); }   // acquire in ctor
    ~Lock() { m_.unlock(); }                              // release in dtor
};

void f(std::mutex& m) {
    Lock guard{m};        // locked here
    doWork();             // even if doWork THROWS...
    // more work...
}                         // ...guard's destructor runs on the way out → unlocks. Always.
```

Read the payoff carefully: there is **no `unlock()` call in `f`**. You can't forget it, can't skip it
on an early return, and an exception can't leak the lock — because unlocking isn't something `f`
does, it's something `guard`'s destruction does, automatically, on every exit.

### How the guarantee works in memory — the unwinding timeline

The magic is stack unwinding. Locals are constructed top-to-bottom and destroyed **bottom-to-top**
(reverse order), and this happens whether you fall off the end, `return`, or `throw`:

```
void f(std::mutex& m) {
    Lock guard{m};        // [1] ctor runs: m.lock()      ── stack grows ──▶
    Buffer buf(1000);     // [2] ctor runs: new int[1000]
    doWork();             //     throws!  ─────────────┐
    ...                                                │
}                                                      │
                                                       ▼
   STACK UNWINDING (exception propagates out of f):
      destroy buf   →  ~Buffer(): delete[]   (freed, no leak)   ◀── reverse order
      destroy guard →  ~Lock():   m.unlock() (unlocked, no deadlock)
   ...exception continues to f's caller, both resources already released.
```

The compiler emitted the destructor calls for you at *every* exit edge. That is the whole reason
RAII and exceptions are a matched pair — Module 14 revisits this from the exception side.

That's the entire idiom. Every RAII type is just "acquire in constructor, release in destructor." The
ones you'll use constantly are already written for you:

| RAII wrapper | Manages | Releases (in its destructor) |
|---|---|---|
| `std::unique_ptr<T>` | one heap object | `delete` |
| `std::vector<T>` | a dynamic array | `delete[]` |
| `std::string` | a character buffer | frees it |
| `std::lock_guard` / `std::scoped_lock` | a mutex | `unlock()` |
| `std::fstream` | an open file | `close()` |

Whenever you reach for `new`, ask first: *is there an RAII type that already owns this for me?*
Usually yes.

---

## 3. When RAII isn't done for you: managing a raw resource

Sometimes you *are* the person writing the wrapper — a memory pool, a handle around a C API, a custom
allocator, a ring buffer over a mmap'd region. Now your class holds a **raw** resource (a `new`'d
pointer, a file descriptor, an OS handle), and you have to make it behave correctly when it's copied,
moved, and destroyed. That's where the **Rule of Five** comes in.

Let's build such a class and watch what breaks. Start with just a constructor and destructor:

```cpp
class Buffer {
    int* data_;         // raw owning pointer
    std::size_t n_;
public:
    Buffer(std::size_t n) : data_{new int[n]}, n_{n} {}  // acquire
    ~Buffer() { delete[] data_; }                        // release
};
```

So far so good — one buffer, allocated in the ctor, freed in the dtor. The trouble starts the moment
you **copy** it.

### The disaster: what the *default* copy does

If you don't write a copy constructor, the compiler generates one. It does a **member-by-member
copy** — and for a pointer, copying the member just copies the **address** (the number), not the
thing it points at:

```cpp
Buffer a(100);   // a.data_ ─▶ 0xfff00 ─▶ [ 100 ints ]
Buffer b = a;    // compiler-default copy: b.data_ = a.data_  →  0xfff00
```

In memory, the shallow copy looks like this — **two owners, one heap block**:

```
   stack                          heap
  ┌──────────────┐
  │ a.data_ ─────┼──────┐
  │ a.n_ = 100   │      │
  ├──────────────┤      ▼
  │ b.data_ ─────┼──▶ 0xfff00 [ 100 ints ]   ◀── both a and b point HERE
  │ b.n_ = 100   │      ▲
  └──────────────┘      │
     b.data_ ───────────┘   (same address copied verbatim)
```

Now **both** `a.data_` and `b.data_` hold `0xfff00` — they point at the *same* heap array. This is a
**shallow copy**, and it's a time bomb. When both go out of scope:

```
~b runs: delete[] 0xfff00     // frees the array
~a runs: delete[] 0xfff00     // frees it AGAIN → DOUBLE FREE, heap corruption, crash
```

(And if `b` had modified "its" array, it would have silently modified `a`'s too — they're the same
memory.) This is exactly the double-free / dangling bug from Module 3, reintroduced silently by a `=`
you didn't think twice about.

**The core rule this leads to:** the moment your class owns a raw resource, the compiler's default
copy/move/destroy are *wrong*, and you must take control of them. There are **five** such special
member functions. Let's motivate each as a fix.

---

## 4. The five special members, motivated one at a time

### 4.1 Destructor — release the resource

Already have it. This is the piece that makes the class RAII in the first place:

```cpp
~Buffer() { delete[] data_; }
```

Writing this is the *signal* that your class owns something — which is precisely why the compiler's
copy/move defaults can no longer be trusted (they don't know about the ownership this destructor
implies). This is the intuition behind the Rule of Five: *declaring a destructor is you telling the
compiler "I manage a resource," so the compiler steps back from guessing how copies/moves should
behave.*

### 4.2 Copy constructor — a *deep* copy, not a shallow one

To copy a `Buffer` safely, don't copy the pointer — **allocate a new array and copy the contents.**
Now the two objects own separate memory, and each destructor frees its own:

```cpp
Buffer(const Buffer& o) : data_{new int[o.n_]}, n_{o.n_} {  // fresh allocation
    std::copy(o.data_, o.data_ + n_, data_);                // copy the elements
}
```

```
   after a deep copy `Buffer b = a;`
   a.data_ ─▶ 0xfff00 ─▶ [ ints ]
   b.data_ ─▶ 0xaa100 ─▶ [ separate copy of the ints ]   ← no sharing, no double free
```

That's the fix for the disaster above: a **deep copy** gives each object its own buffer, so each
destructor frees a distinct block.

### 4.3 Copy assignment — the same, but for `b = a` on an *already-built* `b`

Copy *construction* builds a brand-new object (`Buffer b = a;`). Copy *assignment* overwrites one
that already exists (`b = a;` where `b` was made earlier). It's trickier because `b` already owns a
buffer you must not leak, and you have to survive two hazards:

- **self-assignment** (`b = b;`) — if you naively `delete[]` your own buffer then copy from it, you
  just freed the thing you're reading (use-after-free).
- **an exception** while allocating the new buffer — you must not leave `b` half-destroyed.

The clean trick that handles both for free is **copy-and-swap**:

```cpp
Buffer& operator=(Buffer o) {   // take the argument BY VALUE — the copy happens HERE
    swap(*this, o);             // swap our guts with the fresh copy's
    return *this;
}                               // 'o' (now holding our OLD buffer) is destroyed here
```

Walk through why this is elegant:

- The parameter is **by value**, so the caller's argument is copied into `o` *using the copy
  constructor you already wrote* (which allocates fresh + deep-copies). If that allocation throws, it
  throws **before** you've touched `*this` — so `*this` is left intact. Exception-safe (the *strong*
  guarantee), automatically.
- `swap` exchanges the pointers of `*this` and `o` — cheap, can't throw. After it, `*this` holds the
  new data and `o` holds our **old** buffer.
- When `o` goes out of scope at the `}`, *its* destructor frees the old buffer. You never wrote a
  `delete[]` here at all.
- Self-assignment just works: `b = b` copies `b` into `o`, swaps, and frees the (now redundant) copy.
  No special-case `if (this != &other)` check needed.

```cpp
friend void swap(Buffer& a, Buffer& b) noexcept {   // the helper copy-and-swap uses
    std::swap(a.data_, b.data_);
    std::swap(a.n_,    b.n_);
}
```

If copy-and-swap feels like a lot at first, that's fine — the takeaway is *why* it exists: it's the
one formulation that is self-assignment-safe **and** exception-safe without fiddly manual checks. It
does cost one extra allocation vs. a hand-tuned reuse-the-buffer version, which is why some
performance-critical code writes the manual version instead — know both.

### 4.4 & 4.5 Move constructor and move assignment — steal instead of copy

Deep copies are correct but expensive (allocate + copy a million ints). When the source is a
**temporary about to die** (an rvalue — Module 4) or something you've explicitly `std::move`'d,
copying is wasteful: just **steal its pointer and null out the source** so its destructor frees
nothing:

```cpp
// 4. Move constructor
Buffer(Buffer&& o) noexcept : data_{o.data_}, n_{o.n_} {  // take o's pointer
    o.data_ = nullptr;  o.n_ = 0;                         // empty the source (Module 4)
}

// 5. Move assignment — like move ctor, but free our own buffer first
Buffer& operator=(Buffer&& o) noexcept {
    if (this != &o) {           // guard against b = std::move(b)
        delete[] data_;         // free what we currently hold
        data_ = o.data_;  n_ = o.n_;   // steal o's
        o.data_ = nullptr;  o.n_ = 0;  // empty the source
    }
    return *this;
}
```

The pointer-steal, in memory — **no allocation, no element copy, just three pointer writes**:

```
before move:  a.data_ ─▶ [heap 1000 ints]      b.data_ = ? (uninitialized)
              a.n_ = 1000

Buffer b = std::move(a);

after move:   a.data_ = nullptr                b.data_ ─▶ [heap 1000 ints]   (SAME block, no copy)
              a.n_ = 0                          b.n_ = 1000
              (a's dtor now delete[] nullptr → harmless no-op)
```

This is the shallow-copy-plus-null-out from Module 4 — the thing a raw shallow copy got *wrong* is
now correct precisely *because* we empty the source, so only one object ends up owning the buffer.

### The Rule of Five, stated

**If you write (or `=delete`) any one of these five — destructor, copy ctor, copy assign, move ctor,
move assign — you almost certainly need to handle all five.** They're a set: providing a destructor
(you own something) but leaving the default copy means copies still shallow-copy and double-free.
Getting one right and forgetting the rest silently reintroduces the exact bug we started with.

> **Why `noexcept` on the moves?** `std::vector`, when it grows and must relocate its elements, will
> only **move** them (fast) if their move constructor is `noexcept` — otherwise it falls back to
> **copying** them (slow) to preserve its *strong exception guarantee* (if a move threw halfway
> through a reallocation, the vector couldn't roll back, so it refuses to risk it). A missing
> `noexcept` on your move can silently turn every `push_back`-triggered growth into O(n) deep copies.
> Classic interview question, and it genuinely matters for performance. See §6/§7.

---

## 5. Rule of Three, Rule of Zero, and being explicit

### The Rule of Three (the pre-C++11 version)

Before move semantics existed (C++98/03), there were only **three**: destructor, copy constructor,
copy assignment. Same reasoning, minus the two move members. You'll still see it in older code; the
Rule of Five is just the Rule of Three plus moves.

### The Rule of Zero — what you should actually aim for

Here's the punchline that makes all of the above rare in practice: **don't manage raw resources
yourself.** If your members are already RAII types, the compiler-generated five are all *correct*,
because each member cleans up, copies, and moves itself:

```cpp
class Buffer {
    std::vector<int> data_;   // owns the memory AND handles all five correctly
public:
    explicit Buffer(std::size_t n) : data_(n) {}
    // No destructor. No copy/move. The defaults are right. THIS is the Rule of Zero.
};
```

Compare this to the 30-line raw-pointer version above — same behavior, none of the hazards, because
`std::vector` already got the Rule of Five right *once*, and you're reusing it. Copying this `Buffer`
deep-copies (vector's copy ctor does), moving it steals (vector's move ctor does), destroying it frees
(vector's destructor does) — all for free.

**So the real guidance is:** aim for the **Rule of Zero** — hold `vector`/`unique_ptr`/`string` and
write none of the five. Reach for the **Rule of Five** only when you're writing the low-level wrapper
that *no existing type provides* (a custom pool, a handle around a C API, an allocator). In HFT you do
occasionally write those, which is why you must understand both — but even there, most of your code
should be Rule-of-Zero.

### `=default` and `=delete` — being explicit about the five

You can tell the compiler "generate the default for me" (`=default`) or "ban this operation entirely"
(`=delete`):

```cpp
class NonCopyable {
public:
    NonCopyable() = default;                              // give me the default ctor
    NonCopyable(const NonCopyable&) = delete;             // banning copy → compile error if tried
    NonCopyable& operator=(const NonCopyable&) = delete;
};
```

`=delete` turns a misuse into a **compile-time error** instead of a runtime bug. It's how you express
"this type must never be copied" — exactly right for a pool, a socket, or a lock, where a silent copy
would be a correctness disaster.

> **Subtle gotcha:** declaring *any* copy operation (or a destructor) **suppresses** the implicit
> generation of the move operations. So a class with a user-declared destructor and no move members
> is still copyable but **not movable** — every "move" silently falls back to a copy. If you write a
> destructor and want moves, you must `=default` (or write) them explicitly. This trips up a lot of
> people who "wrote a destructor for logging" and then wonder why their type got slow.

---

## 6. Common pitfalls & UB

- **Shallow copy of an owning pointer → double free** (the whole motivating disaster of §3).
- **Missing `noexcept` on a correct move** → containers silently deep-copy instead of moving; huge,
  invisible performance regression.
- **User-declared destructor silently disables moves** → your type becomes copy-only without you
  noticing (§5).
- **Self-assignment in a hand-written copy/move assign** without a guard → use-after-free
  (`delete[] data_;` then read from the just-freed source). Copy-and-swap sidesteps this.
- **Throwing destructor** → if a destructor throws during stack unwinding, `std::terminate` is called.
  Destructors are implicitly `noexcept`; keep them that way.
- **Half-and-half rule violation:** writing a copy ctor but leaving default copy assign (or vice
  versa) — they must be consistent or you get one deep and one shallow path.
- **Using a moved-from object as if it still owns its resource** — after `std::move(x)`, `x` is
  *valid but unspecified*; only destroy it or assign a fresh value to it (Module 9).

---

## 7. Quiz — tough problems

**Q1.** What does this print, and is there a bug?

```cpp
struct S {
    int* p;
    S(int v) : p(new int(v)) {}
    ~S() { delete p; }
};
int main() {
    S a(5);
    S b = a;              // (!)
    std::cout << *b.p;
}                         // end of main
```

**Answer:** Prints `5`, then **double-free / crash** at scope exit. `S` has no copy ctor, so `S b =
a;` shallow-copies the pointer — `a.p` and `b.p` hold the same address. `~b` deletes it, then `~a`
deletes it again → UB. Fix: deep-copy in a copy ctor (and follow the Rule of Five), or hold a
`std::unique_ptr<int>`/`std::vector`/value member.

---

**Q2.** Your class compiled and worked, then you added `~Widget() { log("destroyed"); }` for
debugging. Suddenly a `std::vector<Widget>` got much slower to grow. Why?

**Answer:** Declaring the destructor **suppressed the implicitly-generated move constructor**. Now
`Widget` is copy-only, so every vector reallocation copies each element instead of moving it. Fix:
`Widget(Widget&&) = default; Widget& operator=(Widget&&) = default;` (and the copies, per Rule of
Five). This is the flip side of the `noexcept` issue and just as sneaky.

---

**Q3.** Two candidates write move assignment. Which is correct, and what's wrong with the other?

```cpp
// A
Buffer& operator=(Buffer&& o) noexcept {
    delete[] data_;
    data_ = o.data_; n_ = o.n_;
    o.data_ = nullptr; o.n_ = 0;
    return *this;
}
// B  (copy-and-swap style)
Buffer& operator=(Buffer o) noexcept { swap(*this, o); return *this; }
```

**Answer:** **B** is safe for both copy and move (one operator handles both; move-constructs `o` from
an rvalue argument, copy-constructs from an lvalue). **A** is broken on **self-move**
`b = std::move(b)`: it `delete[] data_`, then sets `data_ = o.data_` which is the just-freed pointer →
use-after-free / double-free. A needs `if (this != &o)`. (B is slightly slower for the copy case
because it always allocates; trade-off noted in §4.3.)

---

**Q4.** Is `noexcept` on the move constructor a correctness requirement or a performance one? Explain
precisely.

**Answer:** **Performance / behavioral**, not correctness of the move itself. A non-`noexcept` move
still moves correctly when *you* call it. But `std::vector` (and other containers) inspect
`std::is_nothrow_move_constructible` when reallocating: if the move can throw, they **copy** elements
instead of moving, to preserve the strong exception guarantee. So the missing `noexcept` doesn't break
your move — it makes the standard library refuse to *use* it in the one place (relocation) where it
matters most.

---

**Q5.** How many heap allocations happen here, and how many `delete[]`s?

```cpp
Buffer a(1000);            // (1)
Buffer b = a;              // (2)
Buffer c = std::move(a);   // (3)
b = c;                     // (4)  assume copy-and-swap operator=
```

**Answer:** Allocations: (1) allocates 1000 ints; (2) copy-ctor allocates a second block; (3) is a
**move**, no allocation (steals `a`'s block, nulls `a`); (4) copy-and-swap by-value copy allocates a
third block, swaps, then frees `b`'s old block. So **3 allocations**. `delete[]`s at end: `~a`
(nullptr, no-op), `~b`, `~c`, plus the one inside (4) = **3 real frees + 1 no-op** — balanced, no
leak, no double free.

---

**Q6.** Why does copy-and-swap take its parameter **by value** rather than by `const&`? Wouldn't
`const&` avoid a copy?

**Answer:** By-value is deliberate: the copy that must happen anyway is pushed to the parameter
boundary, where the compiler can **elide** it or **move** into it if the argument is an rvalue. That's
what lets *one* by-value `operator=` serve as both copy assignment (lvalue arg → copied into the
param) and move assignment (rvalue arg → moved into the param). A `const&` parameter would force a
copy inside the body and couldn't absorb moves.

---

**Q7.** After `std::vector<int> w; auto v = std::move(w);`, is it legal to call `w.push_back(1)`?

**Answer:** Yes. A moved-from standard-library object is **valid but unspecified** — you can't assume
its contents, but all operations with no precondition on the value (like `push_back`, `clear`,
`size`, assignment) are legal. `w.front()` would be UB (empty precondition), but `push_back` puts it
back into a known-good state. (For your *own* Rule-of-Five types you must design the moved-from state
to be safely destructible/assignable — §4.4 nulls the pointers exactly for this.)

---

**Q8.** Spot the leak:

```cpp
class Conn {
    Socket* s_;
public:
    Conn() : s_(new Socket) {}
    Conn(const Conn& o) : s_(new Socket(*o.s_)) {}     // deep copy
    ~Conn() { delete s_; }
    // no assignment operators written
};
Conn a, b;
b = a;   // <-- ?
```

**Answer:** `b = a;` uses the **compiler-generated copy assignment**, which shallow-copies the
pointer: `b.s_ = a.s_`, **leaking** `b`'s original `Socket` and setting up a double-free. The author
wrote a deep copy *constructor* but forgot copy *assignment* — a Rule-of-Three violation. Add
copy-and-swap `operator=` (or `=delete` copies if a connection shouldn't be copyable).

---

## 8. Indian HFT interview questions

Questions in this shape recur across Tower Research (Gurgaon), Optiver, IMC, Graviton, Quadeye,
AlphaGrep, WorldQuant, Da Vinci, Squarepoint, Jump, HRT, NK Securities, and Mansard — often as a live
whiteboard "write a RAII string/buffer class" followed by drilling.

**Q1. "Write a `unique_ptr` from scratch."**
*Model answer:* A template holding a raw `T*`, a constructor that takes ownership, a `~dtor` that
`delete`s it, **deleted copy ctor and copy assign** (unique = non-copyable), a **`noexcept` move ctor**
that steals the pointer and nulls the source, a **`noexcept` move assign** (delete current, steal,
null — with self-move guard or via swap), plus `operator*`, `operator->`, `get()`, `release()`,
`reset()`. Mention that `reset()` must set the member *before* deleting the old value to be
reentrancy-safe. This single question exercises the entire Rule of Five.

**Q2. "You wrote a move constructor. Why must it be `noexcept`, and what breaks if it isn't?"**
*Model answer:* `std::vector::push_back`/`resize` reallocate; they move elements only if the move is
`noexcept`, else they copy (strong guarantee). Missing `noexcept` → silent O(n) copies on every
growth. It doesn't break correctness, it destroys performance — which in HFT is a correctness bug in
spirit. (Follow-up they like: "how does the container *check*?" → the `std::move_if_noexcept` /
`is_nothrow_move_constructible` trait.)

**Q3. "Explain copy-and-swap. Why is it self-assignment- and exception-safe without any `if`
checks?"**
*Model answer:* Parameter is by value → the copy (and its potentially-throwing allocation) happens
*before* you mutate `*this`; if it throws, `*this` is untouched (strong guarantee). `swap` is
`noexcept`. Self-assignment copies into the parameter first, so there's no aliasing hazard. Cost: one
extra allocation vs. a hand-tuned reuse; mention you'd write the manual version on a hot path.

**Q4. "I declared a destructor. Is my class still movable?"**
*Model answer:* No — a user-declared destructor suppresses implicit move generation, so the class
becomes copy-only, and "moves" silently degrade to copies. You must `=default` or write the moves.
This is a favorite gotcha because it silently regresses performance.

**Q5. "How does C++ avoid memory leaks without a garbage collector, and why does HFT care?"**
*Model answer:* RAII + deterministic destruction: resources release at scope exit, guaranteed even
through exceptions, at a *predictable* instant you control. HFT cares because there are **no GC
pauses** — no unpredictable multi-microsecond stalls in the middle of a trade. Deterministic teardown
is a core reason the domain uses C++.

**Q6. "What is a moved-from object's state? Can I use it?"**
*Model answer:* Valid but unspecified. You may destroy it or assign it a new value; you must not rely
on its contents or call operations with value preconditions. Design your own types' moved-from state
to be a safe empty (nulled pointers, zero size) so destruction is a no-op.

**Q7. "When would you deliberately delete the copy operations?"**
*Model answer:* For types that own a unique, non-duplicable resource — a memory pool, a socket, a
lock, a hardware handle — where a silent copy would be a correctness disaster (two pools over one
slab, two owners of one fd). `=delete` turns the misuse into a compile error.

**Q8. "Rule of Zero vs Rule of Five — which should most of your code be, and why?"**
*Model answer:* Rule of Zero: compose RAII members (`vector`/`unique_ptr`/`string`) and write none of
the five, so there's nothing to get wrong. Reach for Rule of Five only in the small set of low-level
wrappers no existing type provides. Less code = fewer bugs; the standard library already got the hard
part right once.

---

## 9. In the order book

- `OrderPool` (Module 3) is a **Rule-of-Five** type: it owns a big buffer, must **not** be copied
  (`=delete` the copy operations — two pools sharing one slab would be chaos), and can be moved
  (`noexcept` move so a `std::vector<OrderPool>` or a resize wouldn't copy the slab). It's exactly the
  low-level wrapper where you write the special members by hand.
- Everything above it — `Book`, `Limit` — follows the **Rule of Zero**: they hold `std::vector`s or
  references/indices into the pool, so the compiler-generated special members are already correct and
  you write none of them.
- The matching thread relies on RAII's *deterministic* teardown: no GC, so no surprise pause while a
  quote is in flight. Locks around any shared state (rare — the engine is mostly single-threaded) use
  `std::scoped_lock` so an early return or a thrown parse error can never leak a lock and deadlock the
  book.

That's the layering you want: a *small* amount of hand-written Rule-of-Five at the bottom,
Rule-of-Zero everywhere above it.

---

## Key takeaways

- **RAII** = acquire in the constructor, release in the destructor; scope-exit (including exception
  unwinding) makes release unforgettable. It's how C++ avoids leaks without a GC, deterministically.
- The moment a class owns a **raw resource**, the compiler's default copy/move/destroy are wrong
  (shallow copy → double free). That triggers the **Rule of Five**.
- **Copy = deep copy; move = steal-the-pointer-and-null-the-source.** Copy-and-swap gives
  self-assignment- and exception-safety for free (at one extra allocation).
- **`noexcept` on moves is mandatory in practice** — containers copy instead of move without it. A
  user-declared destructor silently disables move generation.
- **Rule of Zero is the default**: compose RAII members and write none of the five. Rule of Five only
  for the rare low-level wrapper. `=delete` to forbid copies; `=default` to opt back in.

**Next:** [08 — Operator overloading & value semantics](08-operator-overloading.md)
