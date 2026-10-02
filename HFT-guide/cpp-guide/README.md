# C++ for HFT — learning roadmap

This track builds one mental model at a time: first the language and object lifetime, then ownership
and generic programming, then hardware and concurrency. Read in order. Later modules deliberately
reuse the vocabulary, memory diagrams, and order-book design decisions established earlier.

## Part I — Language and object model

1. [C++ foundations](01-cpp-foundations.md) — build pipeline, types, initialization, casts, enums,
   storage duration, linkage, and attributes.
2. [Memory and object lifetime](02-memory-lifetime.md) — stack, heap, scope, lifetime, and dangling
   objects.
3. [Pointers and references](03-pointers-references.md) — indirect access, nullability, aliasing, and
   parameter choices.
4. [Dynamic memory](04-dynamic-memory.md) — `new`/`delete`, allocation failure modes, and object
   pools.
5. [Value categories](05-value-categories.md) — lvalues, rvalues, temporary materialization, and the
   intuition behind moving.
6. [`const` and compile-time evaluation](06-const-constexpr.md) — `const`, `constexpr`, `consteval`,
   and `constinit`.
7. [Classes](07-classes.md) — invariants, constructors, destructors, layout, and access control.
8. [Inheritance and virtual functions](08-inheritance-virtual.md) — dynamic polymorphism, vtables,
   RTTI, casts, slicing, and composition.

## Part II — Ownership, value types, and the standard library

9. [RAII and the Rule of Zero/Five](09-raii-rule-of-five.md) — resource ownership and special member
   functions.
10. [Operator overloading](10-operator-overloading.md) — predictable value semantics and strong
    domain types.
11. [Move semantics](11-move-semantics.md) — rvalue references, forwarding, elision, and `noexcept`.
12. [Templates and concepts](12-templates-concepts.md) — deduction, specialization, traits,
    variadics, constraints, and compile-time dispatch.
13. [STL containers and algorithms](13-stl.md) — container families, iterator invalidation,
    algorithms, adapters, vocabulary types, and cache trade-offs.
14. [Smart pointers](14-smart-pointers.md) — exclusive/shared ownership, observers, control blocks,
    and custom deleters.
15. [Exceptions and error handling](15-exceptions-errors.md) — guarantees, `noexcept`, predictable
    status returns, and failure boundaries.

## Part III — Low-latency systems C++

16. [Memory and cache](16-memory-cache.md) — hierarchy, cache lines, layout, alignment, false
    sharing, and data-oriented design.
17. [Concurrency fundamentals](17-concurrency-fundamentals.md) — threads, mutexes, condition
    variables, deadlock, and spinlocks.
18. [Atomics and lock-free basics](18-atomics-lockfree.md) — CAS, memory ordering, ABA, and a bounded
    SPSC ring buffer.
19. [Zero-cost abstraction](19-zero-cost-crtp.md) — static dispatch, CRTP, policy design, branch
    elimination, and measured hints.
20. [C++20 features that matter](20-cpp20-features.md) — concepts, `<bit>`, `span`, ranges,
    comparison, `jthread`, and the project's defensible C++20 story.

## Study rule

For each module, be able to do three things without notes:

1. draw the relevant memory/object/thread picture;
2. state the correctness rule and the common undefined-behavior trap;
3. explain the order-book design decision and its latency trade-off.

That combination is more useful in an HFT interview than memorizing a catalogue of syntax.
