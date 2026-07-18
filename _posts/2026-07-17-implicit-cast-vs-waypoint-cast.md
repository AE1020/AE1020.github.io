---
title:  "Implicit Casts vs the (maybe `(void*)`) Waypoint Cast"
layout: post
---

> *This is my first piece of AI-assisted technical writing.  Rather than talk
> here about giving credit to which-LLM-did-what (and date the C++ content
> itself to the AI boom of 202X) I put some thoughts in another article:*
>
> <https://ae1020.github.io/ai-writing-when-to-stop/>

---

I found myself having to explain why I kept writing `(T*)(void*)ptr` in a
C++ library. I was convinced the `(void*)` hop was a deliberate trick — a way
to force a conversion through the "correct" path instead of an accidental
wrong one. I wrote up why I thought it worked. Then, while getting the
writeup reviewed, it became clear the trick doesn't work in general: it can
silently reproduce the exact pointer-corruption bug it was supposed to
avoid. So I replaced it with a cleaner-looking `implicit_cast<T>()` helper
that sidesteps the problem by construction, and thought I could move on.

Then I tried rebuilding the actual library the idiom came from, and it didn't
compile — not "silently wrong," a hard compiler error. My codebase's real
use case turned out not to be the one I'd been writing about at all. It was
the *mirror image* of it, and the mirror image doesn't have the same fix.
It doesn't even have the same kind of bug.

This turns into two related but genuinely different problems, both hiding
behind the same-looking C-style cast — and answering "is the `void*`
version still an implicit cast?" turns out to need a real answer, not a
slogan.

## Case 1: A Wrapper Hiding the Wrong Overload (Upcasts)

Here's the scenario I originally had in mind — a type with two conversions,
where a direct cast picks the wrong one.

### The trap: a type with two conversions

Suppose you have a small wrapper class that knows how to convert to its
"natural" pointer type implicitly, and — as an escape hatch for generic code
— also offers an explicit, templated conversion to *any* pointer type:

```cpp
template<typename T>
class Wrapper {
    T* p;
  public:
    Wrapper(T* p) : p(p) {}

    operator T*() const { return p; }              // implicit, exact type

    template<typename U>
    explicit operator U*() const {                  // explicit, any type
        return reinterpret_cast<U*>(p);
    }
};
```

The implicit operator is the "happy path" — it fires automatically and
returns the exact pointer you'd expect. The templated explicit operator
exists so that generic code (templates that don't know `T` at the call site)
can still coerce the wrapper to *some* related pointer type when asked
directly.

Now say `Derived` inherits from `Base`, and you have a `Wrapper<Derived>`.
You want a `Base*` out of it. The obvious thing to write is:

```cpp
Base* b = static_cast<Base*>(wrapper);   // looks right...
```

This compiles. It also silently does the wrong thing. Overload resolution
has no `operator Base*()` to reach for, so it falls back to the *only*
conversion that can produce a `Base*`: the templated explicit operator, with
`U = Base`. That operator uses `reinterpret_cast`, which does not perform
base-class pointer adjustment. If `Derived` has a non-trivial layout
relationship with `Base` — a second base class, a virtual base, or just a
compiler that puts things in a different order than you assumed — the
resulting `Base*` can be a corrupted address, not the address `static_cast`
would have produced from an actual `Derived*`.

The bug is invisible at the call site. Nothing warns you. It compiles clean
and often "works" in simple single-inheritance layouts purely by accident of
address arithmetic being a no-op there.

### The Tempting Counter-Attack (And Why It Fails)

The "obvious" fix looks almost too simple:

```cpp
Base* b = (Base*)(void*)wrapper;
```

The reasoning sounds solid — but it rests on a claim about overload
resolution that isn't actually true, and it's worth pinning down now,
because the same mistake comes back with real teeth in Case 2.

A C-style cast to `void*` is direct-initialization, and direct-
initialization considers *every* conversion function, explicit or not.
Explicitness only decides whether a candidate is in the running at all — it
carries no weight once ranking starts. Ranking runs purely on the
conversion sequence from each candidate's *return type* to the target, and
here the templated explicit operator has an edge the implicit one doesn't:
matching its pattern `U*` against the target `void*` lets the compiler
deduce `U = void` directly, so it returns `void*` exactly — an Identity
match. The implicit operator returns `Derived*`, which still needs an
ordinary `Derived* -> void*` conversion to finish the job — ranked
Conversion, strictly worse than Identity. So the *explicit* template
operator, not the implicit one, is actually the candidate that wins this
contest.

It happens not to matter here. Converting *any* object pointer to `void*`
never adjusts the address — the standard requires the value to pass
through unchanged, no matter which conversion function produced the
pointer being converted. So whichever operator actually fires, `(void*)wrapper`
ends up holding the same bits either way: the unadjusted address of the
`Derived` object. Hop one lands on the right address by coincidence, not
because "implicit beats explicit" — a belief that's about to stop being
harmless.

It's hop two that quietly ruins everything: `(Base*)(void* value)`.

A C-style cast from `void*` to a typed pointer behaves like `static_cast`.
And `static_cast<Base*>` of a `void*` is *not* the same operation as
`static_cast<Base*>` of a `Derived*`. The standard only lets a `void*`
round-trip back into the exact object it pointed to (or a pointer-
interconvertible one — informally, an object at the *same address*, such as
the first base in a standard-layout type with no vtable). Outside of that
narrow case, the wording is blunt: **the pointer value is unchanged by the
conversion.**

Once the pointer has been erased to `void*`, the compiler no longer knows
the static type of the thing it came from. It has no class hierarchy to
consult, no offset to apply — it just hands back the same bits it was
given. If `Base` lives at a non-zero offset inside `Derived` (a second base
in multiple inheritance, a virtual base, anything where the base subobject
isn't at the start of the object), then `(Base*)(void*)wrapper` produces a
pointer to the *start of the Derived object*, not to the `Base` subobject.
That is precisely the corrupted address the templated `explicit operator
U*()` produced with its internal `reinterpret_cast` — the bug is back,
just hiding behind an extra pair of parentheses that *looked* principled.

The failure is worse than the original one in one respect: it's much
harder to spot in review. `reinterpret_cast` at least announces "I am doing
something dangerous." `(void*)` announces nothing. It reads like a
type-erasure no-op, and for the single-inheritance, no-vtable case it
*is* a no-op — which is exactly why the bug can live in a codebase for a
long time before a multiple-inheritance or virtual-inheritance type walks
in and breaks it.

### The True Fix: Force an Implicit-Only Context

The actual problem was never "the type gets erased." It's that
`static_cast` and C-style casts perform **direct-initialization**, and
direct-initialization is the context in which that templated `explicit
operator U*()` is eligible to be chosen at all. Explicit conversion
functions are excluded only from **copy-initialization** — the kind of
implicit conversion that happens when a value is used to initialize a
parameter or a `return`, with no cast in sight.

So instead of trying to out-maneuver overload resolution with a detour
through `void*`, force the conversion into a context where the explicit
operator was never a candidate in the first place:

```cpp
template<typename T>
struct type_identity { using type = T; };

template<typename T>
T implicit_cast(typename type_identity<T>::type val) {
    return val;
}
```

(If your standard library has `std::identity`, or you're on C++20, you can
use that instead of hand-rolling it.)

The `typename type_identity<T>::type` wrapping around the parameter type is
what makes this work: it puts `T` in a **non-deduced context**, so the
compiler can't infer `T` from the argument. You must supply it explicitly
at the call site:

```cpp
Base* b = implicit_cast<Base*>(wrapper);
```

Now trace what overload resolution actually does:

1. `T` is fixed to `Base*` before the parameter type is even considered for
   deduction — there's no deduction left to do.

2. The parameter `val` is of type `Base*`, and `wrapper` needs to
   *copy-initialize* it. This is not a cast; it's the same kind of implicit
   conversion that happens when passing an argument to any ordinary
   function.

3. Because it's copy-initialization, the templated `explicit operator
   U*()` is not in the candidate set at all — explicit conversion functions
   are never considered for copy-initialization, full stop.

4. The only viable conversion function left is the implicit `operator
   Derived*()`. The compiler uses it, producing a real `Derived*`.

5. That `Derived*` then needs to become a `Base*` to finish initializing
   `val` — and *this* step is an ordinary derived-to-base pointer
   conversion, the same one `static_cast<Base*>(a_derived_ptr)` would do.
   The compiler has the full type information at this point, so it applies
   whatever offset `Base` actually needs, correctly, in every inheritance
   configuration.

No detour through `void*`, no erased type, no reliance on address
coincidences. The pointer adjustment happens exactly where the compiler has
enough information to get it right.

There's a load-bearing assumption baked into all of this, though: step 5
only works because `Derived* -> Base*` is something the language already
knows how to do *implicitly*. `implicit_cast<T>()` doesn't manufacture a
conversion out of nothing — it just clears an explicit-only competitor out
of the way so that whatever implicit path already exists gets used. If no
implicit path exists at all — which is exactly the situation in Case 2 —
there's nothing left for it to fall back on, and it stops compiling
entirely.

## Case 2: When There's No Implicit Path At All (Downcasts)

Here's the case that actually put `(T*)(void*)expr` in my code in the first
place, and it's the mirror image of Case 1: instead of getting a `Base*`
out of a `Derived`-flavored wrapper, I needed to get a `Derived*` out of a
`Base`-flavored wrapper.

That one-word difference changes everything, because of a rule every C++
programmer learns early and then mostly forgets about: `Derived* -> Base*`
is a *standard, implicit* conversion — the compiler will do it with no cast
at all. `Base* -> Derived*` is **never** implicit, for any hierarchy, under
any circumstances. It's a downcast: you're asserting something the static
type system can't verify on its own ("I happen to know this `Base` is
really a `Derived`"), so the language requires you to say so explicitly,
every time.

Now put a wrapper in the mix. Say you have a small handle type that runs a
check whenever it's converted to its natural pointer type — something like
a debug-only "has this been validated yet" flag:

```cpp
template<typename T>
class Handle {
    T* p;
    mutable bool needs_check = true;
  public:
    Handle(T* p) : p(p) {}

    operator T*() const {                      // implicit, exact type
        if (needs_check) {
            run_debug_validation(p);            // <- the side effect
            needs_check = false;
        }
        return p;
    }

    template<typename U>
    explicit operator U*() const {               // explicit, any type
        return reinterpret_cast<U*>(p);          // <- no side effect at all
    }
};
```

Suppose `Handle<Base>` needs to hand back a `Derived*` (some caller has
verified, out of band, that the underlying object really is a `Derived`).
Reach for the tool that supposedly always works:

```cpp
Derived* d = implicit_cast<Derived*>(handle);   // does not compile
```

This fails outright — a compiler error, not a silent bug. `implicit_cast<T>()`
copy-initializes its parameter from `handle`, and copy-initialization only
considers *implicit* conversion functions. `Handle<Base>` has exactly one:
`operator Base*()`. But `Base* -> Derived*` isn't a standard conversion, so
there's no way to complete the initialization using only implicit
machinery. Overload resolution comes up empty. `implicit_cast<T>()` isn't
picking the wrong overload here — it has nothing left to pick from at all.

So what does a plain cast do instead?

```cpp
Derived* d = static_cast<Derived*>(handle);   // compiles, but skips the check
```

This compiles, for the same underlying reason it compiled in Case 1:
direct-initialization (what `static_cast` and C-style casts perform) *does*
consider `Handle`'s templated `explicit operator U*()`, with `U = Derived`.
Here, though, it's not competing against anything — it's the *only* viable
candidate on the table. The compiler calls it, gets back
`reinterpret_cast<Derived*>(p)`, and `needs_check` never gets cleared.
`run_debug_validation` never runs. The *address* comes out completely fine
(more on that in a moment) — but the entire point of routing through the
implicit operator, its side effect, has been silently skipped.

This is a genuinely different failure mode from Case 1. Case 1 was about
getting a *corrupted address* from the wrong overload. Case 2 is about
getting a *perfectly correct address* while silently skipping behavior the
implicit operator was responsible for running.

### The Tempting Fix for Case 2 (And Why It Also Fails)

Given Case 1's story, the obvious move is to reach for the same two-hop
idiom that Case 1 just spent a whole section warning about, on the theory
that this time the roles are reversed and it'll actually work:

```cpp
Derived* d = (Derived*)(void*)handle;   // looks right...
```

It isn't. The claim Case 1 debunked — "a C-style cast to `void*` prefers
implicit conversions" — was never true, and here it actually costs
something. `(void*)handle` is direct-initialization, so `Handle<Base>`'s
templated `explicit operator U*()` is just as much a candidate as the
implicit `operator Base*()`. Deducing `U = void` gives the explicit
operator an exact-match return type of `void*`; the implicit operator's
`Base*` needs an extra `Base* -> void*` conversion to get there. Identity
beats Conversion, so `(void*)handle` calls the *explicit* operator — the
one with no side effect at all. `run_debug_validation` never runs,
`needs_check` never clears, and the address comes out fine only because —
same as Case 1 — converting any object pointer to `void*` never needs
adjustment, so the bits are identical no matter which operator produced
them.

This is the exact same bug as Case 1, one level further in: reaching for
`(void*)` on the theory that it selects the implicit conversion function,
when it actually just runs an ordinary overload-resolution contest that the
*explicit* operator happens to win whenever the target is `void*`. Case 1
never noticed, because the winning candidate didn't matter there — both
operators land on the same address. Case 2 isn't so forgiving, because
what's riding on the outcome is a side effect, not just an address.

(If you don't want to take this on faith: put a `printf` in each of
`Handle`'s two operators and watch which one actually fires for
`(void*)handle`. It's the explicit one — confirmed by compiling exactly
this example.)

### The Real Fix for Case 2: Copy-Initialize First, Then Reinterpret

The actual fix doesn't route through `void*` at all — it routes through the
*named* intermediate type instead, using plain copy-initialization:

```cpp
Base* validated = handle;                          // copy-init only
Derived* d = reinterpret_cast<Derived*>(validated); // layout-proven reinterpret
```

**Step 1, `Base* validated = handle;`.** This is copy-initialization, not a
cast — the same context `implicit_cast<T>()` relies on in Case 1.
Copy-initialization doesn't rank explicit candidates worse than implicit
ones; it removes them from the candidate set *unconditionally*, before any
ranking happens at all. There's no `U = void` deduction to win a contest
with, because `Base*` is a concrete named type, not a deduced pattern — the
templated `explicit operator U*()` isn't a candidate here, full stop. The
only conversion function left is `operator Base*()`, so it's the one that
runs: `run_debug_validation` fires, `needs_check` clears, and `validated`
holds a correctly-typed `Base*`.

**Step 2, `reinterpret_cast<Derived*>(validated)`.** Now that `validated`
is a plain, concrete `Base*` — no wrapper, no conversion operators left to
interfere — reinterpreting it as `Derived*` is the same operation the
broken `(Derived*)(void*)handle` attempt was reaching for in its second
hop, with the same requirement: it's only safe because `Base` and `Derived`
are laid out so that no offset adjustment is ever needed (standard-layout,
matching `sizeof`, proven separately via `static_assert`). Routing through
`void*` here would add nothing — `Base*` is already a plain pointer with no
operators of its own left to dodge.

The `void*` hop, in other words, was never doing the job this piece
originally credited it with. That job — soliciting the implicit conversion
function and nothing else — was always copy-initialization's job. `void*`
just happened to look like it was forcing the same outcome, in a case
(Case 1) where the actual winning overload didn't matter to the final
bits. Once the source has a side effect that depends on picking the
*correct* operator, `void*` is the wrong tool for that step —
copy-initialization to a named intermediate type is the only version of
"solicit the implicit conversion" the standard actually guarantees.

## So, Is It Still an "Implicit Cast"?

Given both cases, is "copy-initialize, then reinterpret" fair to call "an
implicit cast"? Sort of — but not in the sense that matters most, and not
for the reason this piece originally gave.

- **What actually forces the implicit path:** copy-initialization —
  assigning or passing the source to a variable or parameter of its own
  natural, concrete type. Copy-initialization unconditionally excludes
  explicit conversion functions from the candidate set, which is the only
  version of "give me the implicit conversion and nothing else" that the
  standard actually guarantees, rather than one that depends on how a
  ranking contest happens to come out.

- **What `(void*)` doesn't reliably do:** force the implicit operator to
  run. A C-style cast to `void*` is direct-initialization, which admits
  explicit conversion functions as full candidates, ranked purely by how
  close their return type is to `void*`. A templated `explicit operator
  U*()` can deduce `U = void` for an exact match, beating an implicit
  operator whose return type still needs an extra conversion to reach
  `void*`. Whether that costs you anything depends entirely on whether
  more than the address rides on the outcome — invisible in Case 1,
  load-bearing in Case 2.

- **What the reinterpret step doesn't do, either way:** produce a value of
  some type the language considers implicitly — or even explicitly —
  reachable from the source. There is no conversion from `Handle<Base>` all
  the way to `Derived*`. That step isn't a conversion the type system
  endorses at all; it's a raw reinterpretation of an address, whose
  correctness is a fact you're required to have established independently,
  elsewhere, about the layout of your types.

The honest mental model is: **"copy-initialize to the implicit type, then
reinterpret."** Calling the whole thing "an implicit cast" is fine as
shorthand once you know what's actually buying you the implicit behavior —
but it's copy-initialization doing that work, not `void*`. `void*` (or a
named `reinterpret_cast` — they're equally valid once the intermediate is a
plain pointer) only shows up in the *second* step, where it's just a
bit-preserving relabeling with no say in which conversion function produced
the value being relabeled.

## When each pattern is worth reaching for

**`implicit_cast` and the copy-init waypoint both work by *excluding*
explicit conversion functions from the candidate set — they just start from
opposite endpoints of the same hierarchy.**

Side by side, using the two example types from above:

```cpp
// Case 1 — Wrapper<Derived> holds a pointer whose "natural" type is
// Derived*.  Base* IS implicitly reachable (Derived* -> Base* always is).
// A raw static_cast picks the WRONG overload; implicit_cast picks the
// right one on purpose.

Base* b1 = static_cast<Base*>(wrapper);        // WRONG: corrupted address
                                                //   (hits explicit operator U*())
Base* b2 = implicit_cast<Base*>(wrapper);      // RIGHT: correct address
                                                //   (forces operator Derived*())
Base* b3 = (Base*)(void*)wrapper;               // WRONG: corrupted address, again
                                                //   (hop 2 can't adjust for offset)

// Case 2 — Handle<Base> holds a pointer whose "natural" type is Base*.
// Derived* is NOT implicitly reachable from Base* (downcasts never are).
// implicit_cast has nothing to select -- it fails to compile.  A raw
// static_cast compiles, but silently skips the validation side effect.
// So does the naive (void*) waypoint -- it hits the SAME explicit
// overload static_cast does, for the same reason: U=void is an exact
// match that beats the implicit operator's Base* -> void* conversion.

Derived* d1 = static_cast<Derived*>(handle);      // COMPILES, SKIPS run_debug_validation
                                                   //   (hits explicit operator U*())
Derived* d2 = implicit_cast<Derived*>(handle);    // FAILS TO COMPILE
                                                   //   (no implicit Base* -> Derived*)
Derived* d3 = (Derived*)(void*)handle;             // WRONG: ALSO skips run_debug_validation
                                                   //   (explicit U=void wins the ranking)
Base* validated = handle;                          // RIGHT: copy-init excludes
                                                   //   the explicit operator outright
Derived* d4 = reinterpret_cast<Derived*>(validated); //   runs run_debug_validation,
                                                   //   correct address (layout-guaranteed)
```

The contrast is the whole lesson, but it's not the one this piece first
reached for. In Case 1, both `static_cast` and the naive `void*` cast
choose the *same* wrong overload — the templated `explicit operator U*()`
— because an implicit `Base*` was never on offer for the wrapper's actual
conversion functions to reach; only bypassing the explicit competitor with
`implicit_cast` gets the address right. In Case 2, `static_cast` and the
naive `void*` cast *also* choose the same overload as each other — for the
same underlying reason, direct-initialization admitting the explicit
template as a candidate and letting an exact-match deduction win. What's
different about Case 2 is that `implicit_cast` can't even compile, because
there's no implicit conversion for it to fall back on — which is exactly
why this piece needed a *different* tool for Case 2: not `void*`, but
copy-initialization to the named intermediate type, which excludes the
explicit candidate the same unconditional way `implicit_cast` does.

The two fixes solve non-overlapping problems, and the deciding question is:
**does an implicit path to the target type exist at all?**

**Reach for `implicit_cast<T>()`** when it does — typically an upcast, or
any case where the target type is genuinely, implicitly reachable, but a
broader explicit/templated conversion on the source type would otherwise
hijack a direct cast:

- Smart-pointer-like wrapper types that support generic coercion for
  template metaprogramming, alongside a normal implicit conversion.

- Any class hierarchy using multiple inheritance or virtual inheritance,
  where pointer identity is *not* just "the same address reinterpreted" —
  base subobjects can live at nonzero offsets. This is exactly the case
  where the `void*` shortcut falls apart and `implicit_cast` earns its
  keep.

It is **not** needed for plain pointers to related classes (`Derived* ->
Base*` already does the right thing with a normal `static_cast`, no wrapper
type or helper required). And it doesn't need `void*` at all — that's the
whole point. The fix isn't about erasing type information; it's about
picking initialization contexts where the compiler's overload resolution
rules already do what you want.

**Reach for the copy-init waypoint** when there's no implicit path to the
final type at all — typically a downcast, or any conversion the type system
will never sanction on its own — but the source type still has an implicit
conversion (to *some* intermediate type) that has a side effect you can't
afford to skip:

- Hand-rolled handle/wrapper types where the "natural," implicit conversion
  does bookkeeping — debug validation, reference counting, poisoning,
  assertion checks — that a bare `reinterpret_cast` escape hatch bypasses
  entirely.

- Any place you're prepared to separately prove (via `static_assert` on
  `std::is_standard_layout` and matching `sizeof`, or an equivalent
  domain-specific guarantee) that the reinterpretation in the final step
  needs no address adjustment. Without that proof, don't reach for this —
  the reinterpret step has no safety net of its own.

- Note that this is *not* the same tool as "cast to `void*` first." A
  literal `(void*)` hop is direct-initialization and admits the same
  explicit competitor `implicit_cast` was invented to dodge — it just
  happens to matter less obviously, because the target is `void*` rather
  than a named pointer type. The waypoint has to be copy-initialization to
  the source's own concrete, named type; only then is the explicit operator
  categorically excluded.

Neither tool is a general-purpose substitute for `static_cast`. Each one
exists to route around exactly one specific way a plain cast's overload
resolution goes somewhere other than where you meant.

## A note on naming both workarounds

Whichever one applies, don't leave it as an unnamed one-off — and don't
reach for a bare `(void*)` "for symmetry" with other casts in the code when
`implicit_cast<T>()` was actually what the situation called for, or vice
versa. Give each helper a name and keep it around as a small, reusable
utility.

For the copy-init waypoint, the honest implementation needs to name the
intermediate type explicitly — there's no way to deduce it generically,
since it's whatever concrete type the source's own implicit conversion
operator happens to produce:

```cpp
template<typename Target, typename Intermediate, typename Source>
Target waypoint_cast(const Source& src) {
    Intermediate intermediate = src;                // copy-init: implicit path only
    return reinterpret_cast<Target>(intermediate);  // layout-proven reinterpret
}

// Derived* d = waypoint_cast<Derived*, Base*>(handle);
```

Or, if you're stuck writing it as a macro for a C-compatible codebase, a
small lambda forces the same copy-initialization context without needing a
separate named helper per call site:

```cpp
#define waypoint_cast(TargetType, IntermediateType, expr) \
    ([](IntermediateType intermediate) { \
        return reinterpret_cast<TargetType>(intermediate); \
    }(expr))
```

Either form makes the name do double duty: it says what the target type is,
and the explicit `Intermediate` parameter documents exactly which implicit
conversion is being solicited, instead of leaving it to be reverse-engineered
from the source type's overload set.

1. **Visibility.** A reviewer sees a named idiom — `implicit_cast<Base*>(x)`
   or `waypoint_cast<Derived*, Base*>(x)` — instead of a cast expression
   whose purpose has to be reverse-engineered from context, or worse,
   misdiagnosed as the *other* pattern's fix (or as the broken literal
   `(void*)` version of itself).

2. **Searchability.** Every call site that depends on "give me the implicit
   conversion and nothing else" — for whichever of the two reasons — is one
   grep away, which matters if the wrapper type's conversion operators ever
   change.

3. **Correctness by construction, where it applies.** Both `implicit_cast<T>()`
   and `waypoint_cast<Target, Intermediate>()` work by *excluding* a category
   of overload from consideration — via copy-initialization — rather than by
   out-guessing which overload will fire or which one a `void*` target
   happens to rank best. `waypoint_cast` doesn't get full correctness for
   free, though — its final `reinterpret_cast` rests on a layout invariant
   you have to keep proving separately, so it's worth a comment at the
   definition site pointing at exactly what guarantees it.

Compilers are very good at doing exactly what the standard says and not
what you meant. When a type has more than one way to become a pointer, the
first question isn't "which cast do I reach for" — it's "does an implicit
conversion to what I want exist at all, or am I trying to force a side
effect out of a conversion I can't actually complete." Those are different
problems, and — as it turns out — they have different fixes.

## Addendum: Whitelisting the Cheap Path

`waypoint_cast<Target, Intermediate>()` is safe by construction, but it has
a real cost at every call site: you have to name `Intermediate` by hand,
and it forces an actual, separate copy-initialization step before the
reinterpret. That's the right price to pay when a wrapper's conversion
operators genuinely disagree — different side effects, or a side effect on
one side and none on the other, the way `Handle` does above.

But that's not the only shape a wrapper's conversion operators come in.
Plenty of wrapper types offer several conversion paths that all just hand
back *the same bits with the same side effects* — an implicit "natural
type" operator and an explicit "generic escape hatch" operator that happen
to agree on everything, because one is a thin pass-through to the other, or
neither does anything but return a field. For those types, it provably does
not matter which conversion function `(void*)` picks, and a bare, single-
step `(Target)(void*)(expr)` is both correct and cheaper to write than
naming an intermediate type at every use.

The catch is that "provably does not matter" is a claim about the *type*,
not about any individual cast expression — and it has to be re-verified by
a human every time that type's operators change. Leaving that judgment
silently implied at each call site is exactly how this article's original
bug got written in the first place. So instead of trusting each call site,
push the claim into a single, explicit, opt-in declaration attached to the
type itself, and have every use of the cheap path check it automatically at
compile time:

```cpp
// Nothing is safe by default -- only raw pointers, which have no competing
// conversion operators to race in the first place.
template<typename T>
struct VoidWaypointSafe : std::is_pointer<T> {};

// Opt a specific wrapper in, immediately next to its own definition, with a
// comment justifying the claim:
//
//   Every conversion path PtrWrapper offers returns the same address with
//   no side effects of its own, so it doesn't matter which one (void*)
//   picks.
//
template<typename T>
struct VoidWaypointSafe<PtrWrapper<T>> : std::true_type {};
```

```cpp
template<typename T>
struct VoidWaypointSafeChecker {
    static_assert(
        VoidWaypointSafe<T>::value,
        "void_waypoint_cast(): T not vouched safe -- see VoidWaypointSafe"
    );
};

// Forces a template to be instantiated (and any static_assert inside it to
// run) purely at compile time, with zero runtime cost -- sizeof() never
// evaluates its operand.
#define DUMMY_INSTANCE(...) static_cast<void>(sizeof(__VA_ARGS__))

#define void_waypoint_cast(TargetType, expr) \
    (DUMMY_INSTANCE(VoidWaypointSafeChecker< \
        typename std::remove_cv<typename std::remove_reference< \
            decltype(expr)>::type>::type>), \
    (TargetType)(void*)(expr))
```

The comma operator is doing the real work here: the left side of `(A, B)`
is evaluated (at compile time only, in this case) purely for its checking
side effect, then discarded, and the expression's value is `B` —
the plain `(void*)` cast, completely unchanged at runtime. `sizeof()` never
evaluates its operand, so `DUMMY_INSTANCE` costs nothing at runtime either;
all it does is force the compiler to instantiate
`VoidWaypointSafeChecker<T>` for whatever `T` the expression actually has,
which is what triggers the `static_assert` — and a failed `static_assert`
is a compile error, not a debug-build check that a release build quietly
skips. Any type that hasn't been (or is no longer) vouched for fails to
compile, by name, at the exact call site that used it — there is no path
where an unvetted type silently reaches the cast.

This is deliberately a whitelist, not a detector: nothing here inspects a
type's operators and decides for itself whether they agree. That judgment
has to be made once, by a person, sitting next to the type's own
definition, as a small and auditable claim ("every conversion path this
type has returns the same address with the same side effects"). Get that
claim wrong and this reintroduces the exact bug the rest of this article is
about — just laundered through a trait that looks like a safety check. So
it earns its keep only when the claim is genuinely trivial to verify by
reading the type once, and it should stay rare and deliberate rather than
becoming the default way of avoiding `waypoint_cast<Target, Intermediate>()`'s
extra typing.
