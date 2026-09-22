# Module 1 — Memory & object lifetime: stack vs heap

Before pointers, before classes, you need a correct model of **where data lives** and **when it
dies**. Nearly every C++ bug (dangling pointers, leaks, use-after-free) and nearly every
performance decision (why the heap is slow, why the cache matters) traces back to this one mental
model. HFT interviews open here on purpose: if you can't draw where a variable lives and say
exactly when it's destroyed, nothing built on top — RAII, move semantics, lock-free queues — will
be solid. This module builds that model from zero.

---

## 1. What "memory" actually is

When your program runs, the operating system gives it a large, flat array of bytes. Every byte has
a numeric **address** — think of memory as one gigantic array `byte mem[...]` where the index *is*
the address. On a 64-bit machine those addresses are 64-bit numbers (which is why a pointer is 8
bytes — it's just an index into that array).

Your process doesn't get one undifferentiated blob, though. The OS and the compiler carve it into
**regions**, each with different rules about lifetime and speed:

```
High addresses
┌───────────────────────────┐
│  Stack                     │  grows DOWN ↓   (local variables, call frames)
│      │                     │
│      ▼                     │
│                            │
│      ▲                     │
│      │                     │
│  Heap (free store)         │  grows UP ↑     (new / malloc / vector storage)
├───────────────────────────┤
│  BSS / Data (static)       │  globals, static locals — live whole program
├───────────────────────────┤
│  Text (code)               │  the machine instructions — read-only
└───────────────────────────┘
Low addresses
```

Four regions matter conceptually; two matter *constantly*:

| Region | Holds | Lifetime | Speed |
|--------|-------|----------|-------|
| **Stack** | local variables, function call frames | automatic (scope-bound) | fastest |
| **Heap** (free store) | objects you allocate explicitly | manual (you decide) | slower, variable |
| Static/global | globals, `static` locals | whole program | — |
| Code/text | the compiled instructions | whole program | read-only |

The stack and the heap grow *toward each other* from opposite ends of the address space. That
layout is why a stack overflow and a heap-exhaustion feel like different failures even though both
are "out of memory."

---

## 2. The stack: automatic storage

The **stack** is a region managed with dead-simple discipline: last-in, first-out, like a stack of
plates. Every time you call a function, the program pushes a **stack frame** — a contiguous block
holding that function's local variables, its parameters, the return address, and saved registers.
When the function returns, the entire frame is popped in one motion.

```cpp
void foo() {
    int x = 5;       // created on the stack
    double y = 3.2;  // created right after x
}                    // <-- x and y DESTROYED automatically here
```

Here's the key insight about *how* allocation happens. The CPU has a **stack pointer** register
(`rsp` on x86-64) that always points at the top of the stack. "Allocating" `foo`'s locals is a
single subtraction — move `rsp` down by the frame size. "Deallocating" on return is a single
addition — move `rsp` back up. That's it. No searching, no bookkeeping, no free-list.

```
                   Before foo()          Inside foo()              After foo() returns
                 ┌──────────────┐      ┌──────────────┐          ┌──────────────┐
   main's frame  │ a: 1         │      │ a: 1         │          │ a: 1         │
                 └──────────────┘      ├──────────────┤          └──────────────┘
                       ▲ rsp           │ ret addr     │                ▲ rsp
                                       │ y: 3.2       │  foo frame
                                       │ x: 5         │
                                       └──────────────┘
                                             ▲ rsp   (rsp moved DOWN by frame size)
```

As calls nest, frames stack up; as they return, frames pop off:

```cpp
int main() {
    int a = 1;   // stack:  [a]
    foo();       // push:   [a][x,y]   ... then pop -> [a]
    return 0;
}                // [a] popped, a destroyed
```

**Why it's fast:** allocating a local is essentially free — one register op, part of reserving the
frame. There is no allocator involved, no lock, no cache miss (the top of the stack is almost
always hot in L1 cache because you just touched it). In HFT you keep everything on the stack when
you possibly can, precisely for this reason.

**Why it's limited:** the stack is a *fixed-size* block reserved when the thread starts —
typically 1 MB on Windows, 8 MB on Linux (`ulimit -s`). It's small on purpose. Two ways to blow it:
deep/infinite recursion (each call adds a frame) and huge local arrays (`int big[10'000'000];` is
40 MB — instant stack overflow). A stack overflow usually doesn't throw; it just crashes (SIGSEGV),
because there's no more room to push a frame.

---

## 3. Lifetime vs scope — the distinction that trips everyone up

These two words sound like synonyms. They are not, and interviewers separate the two deliberately.

- **Lifetime** = the span of time during which the object *exists in memory* — from construction to
  destruction. This is a runtime property.
- **Scope** = the region of *source code* where you can refer to the object by its name. This is a
  compile-time property.

For stack (automatic) variables they coincide almost exactly: enter the block, the object is born
(constructed); leave the block, it dies (destructed). This tight coupling is one of C++'s central
guarantees and the entire basis of RAII (Module 7): tie a resource — heap memory, a file, a lock,
a socket — to a stack object, and the resource is released automatically the instant scope exits,
*even if an exception is thrown while unwinding*.

```cpp
{
    int x = 5;   // lifetime + scope begin
}                // both end here — destructor (if any) runs, storage reclaimed
```

But they can diverge. A `static` local has a lifetime spanning the whole program yet a scope
limited to its function. A heap object (Module 3) has a lifetime you control manually and *no* name
at all — you reach it only through a pointer. Keeping "how long it lives" and "where I can name it"
as separate axes in your head is what lets you reason about dangling pointers and moved-from
objects later.

---

## 4. The trap this prevents: dangling references to locals

Because a stack local dies the moment its scope exits, handing out its address is a time bomb:

```cpp
int* getPointer() {
    int local = 42;
    return &local;   // address of a stack variable
}                    // <-- local DESTROYED here; frame popped

// caller: int* p = getPointer();  // p points into abandoned stack memory -> UB
```

`local` lives inside `getPointer`'s frame. The `return` pops that frame — `rsp` moves back up — and
the memory `p` points at is now free to be reused by the *next* function call. Read or write through
`p` and you have a **dangling pointer**: one of the most common serious C++ bugs, and one that often
*appears* to work (the old value is still sitting there until something overwrites it), which is
what makes it insidious.

```
getPointer() returns &local ──┐
                              ▼
   ┌──────────────┐      p ─► 0x7ff…30  (was `local`, now garbage / reused)
   │ p: 0x7ff…30  │          ▲
   └──────────────┘          └── this slot is BELOW rsp now — no longer owned by anyone
```

**Rule: never return the address or reference of a local variable.** If data must outlive the
function that created it, it belongs on the heap (Module 3) or must be passed in by the caller. This
single rule is *the* reason the heap exists in most programs.

---

## 5. Object initialization (the learncpp detail worth nailing)

C++ has an infamous number of initialization syntaxes. The modern advice is: **prefer brace
initialization** `{}`, because it is uniform and it forbids narrowing conversions.

```cpp
int a;        // default-init: for a built-in at block scope -> INDETERMINATE (garbage). Reading is UB.
int b = 5;    // copy-init
int c{5};     // direct-list (brace) init — preferred; also rejects narrowing (e.g. int c{3.5} won't compile)
int d{};      // value-init -> 0 for built-ins
```

The dangerous one is `int a;`. For a *built-in* type at block scope this does **not** zero it — the
slot holds whatever bytes were left over from a previous stack frame. Reading that value before
writing it is **undefined behavior**, not merely "you get a random number." The optimizer is allowed
to assume you never do this and can miscompile code around it.

```
Stack slot for `a` after `int a;`  →  ┌────────────────┐
                                       │ ?? ?? ?? ??    │  leftover bytes from an earlier call
                                       └────────────────┘
`int d{};`                          →  ┌────────────────┐
                                       │ 00 00 00 00    │  guaranteed zero
                                       └────────────────┘
```

Contrast this with class types that have a default constructor — those *are* initialized by the
constructor even under `int a;`-style syntax. The garbage problem is specifically about built-ins
and trivial types. On the HFT hot path this bites when a POD `Order` struct is left partially
initialized and a stale price/qty leaks through.

---

## Common pitfalls & UB

- **Returning `&local` or `T& local`** — dangling; the #1 stack bug.
- **Reading an uninitialized built-in** (`int x; use(x);`) — UB, not "a random value."
- **Huge stack arrays** (`double m[1'000'000];` as a local) — stack overflow / crash, no diagnostic.
- **Deep or unbounded recursion** — each frame consumes stack; blows past `ulimit -s`.
- **Assuming `static`-local lifetime == scope** — its lifetime is the whole program; only its name is scoped.
- **Dangling reference from a temporary**: `const std::string& s = getName() + "!";` — the temporary
  survives *only* because of lifetime extension; do the same through a function boundary and it dies.
- **Mixing up "destroyed" with "memory zeroed"** — destruction reclaims the *slot*; the bytes are
  usually left as-is until overwritten, which is exactly why use-after-free sometimes "works."

---

## Quiz — tough problems

**Q1.** What does this print, and is it well-defined?
```cpp
int& bad() { int x = 10; return x; }
int main() { int& r = bad(); std::cout << r; }
```
**Answer:** Undefined behavior. `x` lives in `bad`'s frame, which is popped on return; `r` is a
dangling reference. It may print `10`, garbage, or crash. Compilers warn (`-Wreturn-local-addr`).

**Q2.** Does `int arr[3];` at block scope zero-initialize its elements? What about `int arr[3]{};`
and a `static int arr[3];`?
**Answer:** `int arr[3];` at block scope — indeterminate (garbage). `int arr[3]{};` —
value-initialized to `{0,0,0}`. `static int arr[3];` — statics are **zero-initialized** by the
runtime before `main`, so `{0,0,0}`. Storage duration changes the default.

**Q3.** Roughly how much stack does this consume, and what happens?
```cpp
void f() { char buf[16 * 1024 * 1024]; buf[0] = 1; }
```
**Answer:** 16 MB in a single frame. On Linux (8 MB default) this overflows the stack the moment
the frame is entered → SIGSEGV, typically before any code runs. The touch of `buf[0]` is irrelevant;
reserving the frame is what kills it.

**Q4.** Is this safe? Explain precisely.
```cpp
const std::string& greet() {
    std::string s = "hi";
    return s;
}
```
**Answer:** Not safe — dangling reference. Reference return type doesn't extend the lifetime of a
local; `s` is destroyed at `}`. (Lifetime extension only applies when a temporary is bound directly
to a local reference, never across a function return.)

**Q5.** After this block, is the memory that held `x` zeroed?
```cpp
{ int x = 42; }
```
**Answer:** No. Leaving scope ends `x`'s lifetime and frees the slot logically, but the bytes `42`
usually remain until the next frame overwrites them. "Destroyed" ≠ "zeroed." This is why reading a
dangling pointer can still show the old value.

**Q6.** Two threads, each running `f()` which has a local `int counter = 0;`. Do they share
`counter`?
**Answer:** No. Each thread has its **own stack**, so each call frame — and each automatic local —
is per-thread and per-call. Locals are never shared across threads unless you take their address and
publish it. (This is why the SPSC queue in Module 16 must use `std::atomic` for shared state, not a
plain local.)

**Q7.** Predict the output:
```cpp
int* p;
{ int y = 7; p = &y; }
{ int z = 99; std::cout << *p; }
```
**Answer:** UB, but *in practice* often prints `99`. `y` and `z` typically occupy the **same stack
slot** (same offset from the frame base, reused after the first block ends), so `*p` reads whatever
`z` wrote. Never rely on this — it's the textbook demonstration of why dangling pointers are
dangerous rather than merely "empty."

**Q8.** Where does the string *data* live for `std::string s = "hello world this is long";` declared
as a local?
**Answer:** The `std::string` *object* `s` (its pointer/size/capacity, ~24–32 bytes) is on the
stack; the *character buffer* it manages is on the **heap** (for a string long enough to exceed the
Small String Optimization buffer). Short strings live entirely inside `s` on the stack (SSO). This
"handle on stack, payload on heap" split is the theme of Module 3.

---

## Indian HFT interview questions

These are the kinds of stack/heap questions actually posed on India desks — Tower Research Capital
(Gurgaon), Graviton Research Capital (Gurgaon), Optiver, IMC, Quadeye, AlphaGrep, WorldQuant, Da
Vinci, NK Securities, Squarepoint.

**Q. "Walk me through what happens in memory when a function is called and returns."**
*Model answer:* The caller pushes arguments (per the calling convention — often first few in
registers), a return address is pushed, the stack pointer is decremented to reserve a frame for the
callee's locals and saved registers. On return the callee restores registers, the stack pointer is
incremented to pop the frame, and control jumps to the return address. Allocation/deallocation of
locals is just moving `rsp` — no allocator, which is why it's ~free.

**Q. "Why is the stack faster than the heap? Give the mechanism, not just 'it's faster.'"**
*Model answer:* Stack allocation is a single pointer adjustment with no search or synchronization,
and the top of the stack is almost always hot in L1 cache. Heap allocation must search a free-list /
size classes, maintain metadata, may take a lock (shared across threads), and may fault in new pages
via a syscall — all with variable cost. Latency *and* variance are worse. (See Module 3.)

**Q. "A junior returns `&local` from a function. What's wrong, and would it always crash?"**
*Model answer:* It's a dangling pointer — the local's frame is popped on return, so the address
points at reclaimed stack. It usually does *not* crash immediately; the old value often survives
until the next call overwrites that slot, so the bug hides in testing and surfaces in production.
That non-determinism is exactly why it's dangerous.

**Q. "How much stack does a thread get, and why does that matter for a matching engine?"**
*Model answer:* Typically 1–8 MB, fixed at thread creation. It matters because large local buffers
or deep call chains can overflow it silently, and because we deliberately keep hot-path data on the
stack (or in pre-allocated pools) to avoid heap latency — but that budget is finite, so big
per-order state goes in a pool, not a local array.

**Q. "Difference between lifetime and scope? Give an example where they differ."**
*Model answer:* Lifetime is when the object exists at runtime; scope is where the name is visible in
source. They coincide for automatic locals. A `static` local differs: its lifetime is the whole
program, but its name is only visible inside the function. A heap object differs too: lifetime is
manual, and it has no name at all.

**Q. "Is reading an uninitialized `int` just 'a random value'?"**
*Model answer:* No — it's undefined behavior. The compiler may assume it never happens and optimize
accordingly, so the effect can be worse than a garbage value (e.g. a branch on it being both taken
and not taken). Always initialize; prefer `int x{};`.

---

## In the order book

Orders must rest in the book across many function calls — from the moment they arrive until they're
filled or cancelled, which may be seconds or hours later. They therefore **cannot** live on the
stack of the function that received them; that frame is popped almost immediately, and any pointer
into it would dangle. So order storage is heap-backed — and, on the hot path, specifically a
**pre-allocated object pool** (Module 3) rather than per-order `new`, to get heap *lifetime* without
heap *latency*.

Transient things — a loop index, a temporary price computed while walking a price level, a scratch
`Trade` being assembled — stay on the stack, where allocation is free and the data is cache-hot.
Knowing which category a given piece of data falls into (does it outlive this call? is its size
fixed? is it shared?) is the very first design decision for every field in the engine, and it's the
model everything else in this guide builds on.

---

## Key takeaways

- Memory is a flat, addressed byte array carved into regions; **stack** (automatic, fast, small,
  scope-bound) and **heap** (manual lifetime, slower, large) are the two you touch constantly.
- Stack allocation is a single pointer move — no allocator, no lock, cache-hot — which is why HFT
  keeps everything possible on the stack.
- **Lifetime** (when it exists) and **scope** (where you can name it) are different axes; they
  coincide for automatic locals but diverge for statics and heap objects.
- **Never return the address/reference of a local** — the frame is popped on return and the pointer
  dangles; it often "works" in testing, which makes it worse.
- Prefer brace init (`int x{};`); reading an uninitialized built-in is **UB**, not a random value.
- Data that must outlive its creating function belongs on the heap (via an RAII owner) — this is the
  bridge into Module 3.

**Next:** [Module 2 — Pointers & references](02-pointers-references.md) — how you refer to objects
indirectly, the machinery every data structure and the whole order-book graph is built on.
