---
title:  "Implicit Casts vs the `(void*)` Waypoint"
layout: post
---

> *(Credit to Claude.ai for fleshing out my codebase comments on the void
> cast idiom into a full article...to Gemini for reviewing it and catching a
> case where the idiom was technically broken...and then to my own build
> logs for pointing out that while `implicit_cast` fixes upcasts, it
> completely breaks downcasts — proving that a cast's safety depends
> entirely on which direction you're traveling through the class
> hierarchy!)*  😲

I found myself having to explain why I kept writing `(T*)(void*)ptr` in a
C++ library. I was convinced the `(void*)` hop was a deliberate trick — a way
to force a conversion through the "correct" path instead of an accidental
wrong one. I wrote up why I thought it worked. Then, while getting the
writeup reviewed, it became clear the trick doesn't work in general: it can
silently reproduce the exact pointer-corruption bug it was supposed to
avoid. So I replaced it with a cleaner-looking `implicit_cast<T>()` helper
that sidesteps the problem by construction, and moved on.

Then I rebuilt the actual library the idiom came from, and it didn't
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

The reasoning feels solid: a C-style cast to `void*` tries implicit
conversions first, so `(void*)wrapper` is *forced* to go through the
implicit `operator Derived*()` — the templated explicit operator never gets
a chance to run, because the implicit one is strictly preferred. That part
is true. Hop one really does give you the exact, correctly-adjusted
`Derived*`, just erased to `void*`.

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

### The Fix for Case 2: Solicit the Side Effect, Then Reinterpret

Going back to the two-hop `void*` idiom — the one Case 1 seemed to rule out
— turns out to be exactly the right tool here, for a completely different
reason than originally assumed:

```cpp
Derived* d = (Derived*)(void*)handle;
```

**Hop 1, `(void*)handle`.** A C-style cast to `void*` prefers implicit
conversions. `Handle<Base>` has exactly one — `operator Base*()` — so that's
the one that runs. `run_debug_validation` fires, `needs_check` gets
cleared, and the resulting `Base*` converts trivially to `void*`. The
explicit template operator was never a candidate, because an implicit
candidate that fully satisfies the target type is always preferred over an
explicit one.

**Hop 2, `(Derived*)(void* value)`.** This is a bare reinterpretation of the
address bits — the compiler has no class hierarchy left to consult once the
type has been erased to `void*`. Whether that's safe depends entirely on
whether `Base` and `Derived` are laid out so that no offset adjustment is
ever needed — e.g. both are standard-layout, the same size, and derivation
adds no data members, so the "derived" object literally *is* the base
object with a more specific compile-time label on it. That's not something
the language checks for you at this cast; it has to be a fact you've
separately proven about your own type hierarchy (a `static_assert` on
`std::is_standard_layout` and matching `sizeof` is the usual way to pin
that down).

So the `void*` waypoint isn't fixing an address problem here — the address
was never wrong. It's solving a *sequencing* problem: force the one
conversion function that has the side effect you need to be the one that
actually runs, and only afterward, manually reinterpret the result to the
type you actually wanted — leaning on an external guarantee that no bits
need to move to do so.

It's worth naming the reversal explicitly, because `(void*)` is playing two
opposite roles across the two cases:

- **In Case 1 (the upcast), `void*` is a blinder.** It erases exactly the
  type information — *which* base subobject you meant — that a later,
  class-aware cast would have needed in order to adjust the address
  correctly. That's why the naive fix made things worse, not better.

- **In Case 2 (the downcast), `void*` is a firewall.** It doesn't erase
  anything a later cast needed; it erases the *wrapper class itself*,
  along with the side-effect-free `explicit operator U*()` riding along
  with it, so that the only conversion function left standing is the
  implicit one carrying the side effect you actually wanted to run.

Same two hops, same-looking syntax — but the first hop is buying you
completely different things depending on which direction you're travelling
through the hierarchy.

## So, Is It Still an "Implicit Cast"?

Given both cases, is `(T*)(void*)expr` fair to call "an implicit cast"?
Sort of — but not in the sense that matters most.

- **What it does force:** hop 1 genuinely forces the compiler to choose
  whichever conversion function the source type declared *implicit*, over
  any explicit competitor. That's a real, well-defined effect of routing
  through `void*`, and it's the entire reason the idiom does anything at
  all — whether what you cared about was the address that conversion
  produces (Case 1, where it doesn't actually help, because the *second*
  hop throws that correctness away for unrelated types) or the side effect
  it runs (Case 2, where it's exactly what's needed).

- **What it doesn't do:** produce a value of some type the language
  considers implicitly reachable from the source. There is no implicit —
  or explicit — conversion from `Handle<Base>` all the way to `Derived*`.
  The final hop isn't a conversion the type system endorses at all; it's a
  raw reinterpretation of an address, whose correctness is a fact you're
  required to have established independently, elsewhere, about the layout
  of your types.

The honest mental model is: **"solicit the implicit conversion, then
reinterpret."** Calling the whole two-hop expression "an implicit cast" is
fine as shorthand once you know what it actually buys you — but it isn't
one language-level implicit conversion from source to target. It's an
implicit conversion to some *intermediate* type, glued to a manual
reinterpretation that only you can vouch for.

## When each pattern is worth reaching for

**`implicit_cast` is for compiler-accepted conversions; `void_waypoint_cast`
is for recovering a typed pointer from erased storage.**

> Note: Credit to Perplexity.ai for the above sentence.

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

Derived* d1 = static_cast<Derived*>(handle);    // COMPILES, SKIPS run_debug_validation
                                                //   (hits explicit operator U*())
Derived* d2 = implicit_cast<Derived*>(handle);  // FAILS TO COMPILE
                                                //   (no implicit Base* -> Derived*)
Derived* d3 = (Derived*)(void*)handle;           // RIGHT: runs run_debug_validation,
                                                //   correct address (layout-guaranteed)
```

The contrast is the whole lesson: in Case 1, both `static_cast` and the
`void*` waypoint choose the *same* wrong overload — the templated
`explicit operator U*()` — because an implicit `Base*` was never on offer
for the wrapper's actual conversion functions to reach; only bypassing the
explicit competitor with `implicit_cast` gets the address right. In Case 2,
`static_cast` and the `void*` waypoint choose *different* overloads —
`static_cast` takes the only path direct-initialization can see
(`explicit operator U*()`), while the `void*` waypoint's first hop takes the
only path copy-initialization can see (`operator Base*()`), which is
precisely the one carrying the side effect. `implicit_cast` isn't even in
the running for Case 2, because there's no implicit conversion for it to
select in the first place — the failure to compile *is* the signal that
you're looking at a Case 2 problem, not a Case 1 one.

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

**Reach for the `void*` waypoint** when there's no implicit path to the
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
  domain-specific guarantee) that the reinterpretation in the second hop
  needs no address adjustment. Without that proof, don't reach for this —
  the second hop has no safety net of its own.

Neither tool is a general-purpose substitute for `static_cast`. Each one
exists to route around exactly one specific way a plain cast's overload
resolution goes somewhere other than where you meant.

## A note on naming both workarounds

Whichever one applies, don't leave it as an unnamed one-off — and don't
reach for a bare `(void*)` "for symmetry" with other casts in the code when
`implicit_cast<T>()` was actually what the situation called for, or vice
versa. Give each helper a name and keep it around as a small, reusable
utility. For the `void*` one, `void_waypoint_cast<T>()` (or
`void_waypoint_cast(T, expr)` if you're stuck writing it as a macro) seems
like a reasonable name if it doesn't already have a name in your codebase:
it says what the target type is, and the `_cast` suffix marks it as a cast
operation rather than something that merely reads as "make this a `void*`."

1. **Visibility.** A reviewer sees a named idiom — `implicit_cast<Base*>(x)`
   or `void_waypoint_cast<Derived*>(x)` — instead of a cast expression whose
   purpose has to be reverse-engineered from context, or worse,
   misdiagnosed as the *other* pattern's fix.

2. **Searchability.** Every call site that depends on "give me the implicit
   conversion and nothing else" — for whichever of the two reasons — is one
   grep away, which matters if the wrapper type's conversion operators ever
   change.

3. **Correctness by construction, where it applies.** `implicit_cast<T>()`
   works by *excluding* a category of overload from consideration rather
   than by out-guessing which overload will fire, so it stays correct even
   as the class hierarchy underneath it changes shape. `void_waypoint_cast`
   doesn't get this for free — its correctness rests on a layout invariant
   you have to keep proving separately, so it's worth a comment at the
   definition site pointing at exactly what guarantees it.

Compilers are very good at doing exactly what the standard says and not
what you meant. When a type has more than one way to become a pointer, the
first question isn't "which cast do I reach for" — it's "does an implicit
conversion to what I want exist at all, or am I trying to force a side
effect out of a conversion I can't actually complete." Those are different
problems, and — as it turns out — they have different fixes.
