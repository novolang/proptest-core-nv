# proptest-core-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`. Installing this package works;
calling it panics with `not implemented`.

## What this is

Generators and shrinking with **no randomness in them**. A strategy is a
pure function from a recorded choice sequence to a value; the
combinators compose over that; and shrinking is arithmetic on the
sequence, walked by whoever is driving. The half that draws a seed,
runs the cases and reports the failure is
[`proptest-nv`](https://github.com/novolang/proptest-nv), which depends
on this one.

```
novo pkg add proptest-core-nv
novo pkg build
novo test
```

## The one example that will work

```novo
use strategy
use tape

fn reversed_twice(xs: [Int]) -> Bool
    list.reverse(list.reverse(xs)) == xs

fn first_case(seed: Int) -> [Int]
    let g = strategy.list_of(strategy.int_range(0 - 100, 100), 0, 32)
    let (xs, _) = strategy.draw_with(g, tape.from_seed(seed))
    xs
```

The seed is the whole of the input. Two calls with the same seed
produce the same list, on any machine, in any process, with nothing
exported and no clock consulted — which is what makes a failure
reproducible from one integer.

## The load-bearing interface

```novo ignore
pub struct Tape                       // a seeded source that RECORDS what it gave
pub struct PropStrategy<T>
    draw: fn(Tape) -> (T, Tape)

pub fn next(t: Tape, bound: Int) -> (Int, Tape)     // the one primitive draw
pub fn next_try(p: ShrinkPlan) -> ?ShrinkTry        // the walk the runner drives
```

Not a type — a **shape**: a generation is a function of a list of
integers, and that list is kept. Everything else follows.

- **Reproduction** is one integer. The tape holds the seed and the
  splitmix64 step; nothing else moves.
- **Shrinking** is `[Int] -> [Int]`. It works for every strategy at
  once, including a `flat_map` whose later draws depend on earlier
  ones — which is exactly where an integrated shrinker is weakest,
  because the tree it built for the second draw was built for a value
  the first no longer has.
- **The core never calls a property.** `shrink.next_try` hands out a
  candidate and waits for `keep` or `reject`. Calling the property
  would cost whatever the property costs, and a `core` package's budget
  is `[]`.

And one rule every generator here obeys, because the shrinker rests on
it: **a smaller choice means a simpler value.** `int_range` maps choice
0 to `lo`, `bool_any` maps it to `false`, `option_of` to `None`,
`one_of` to the first branch, `list_of` draws its length first.

## Which shrinking model this ports, and why

| | proptest | hypothesis |
| --- | --- | --- |
| a strategy makes | a `ValueTree` | a value from a choice sequence |
| candidates are | the value's own simplifications | shorter or smaller choice sequences |
| shrinking walks | the tree, mutating it | the sequence, replaying it |

**This ports hypothesis's**, and the first two reasons are about this
language rather than about taste.

1. A `ValueTree` needs one named type per combinator and an
   existential to hold a heterogeneous list of them. A trait bound here
   cannot carry a type argument — SPEC § 3.6's grammar is
   `trait_bound ::= TYPE_IDENT [ '[' ident ']' ]`, where the bracket is
   the *effect* argument — so `fn map<A, B, S: Draws<A>>(…)` is a parse
   error and a combinator generic over the value type cannot be written
   against a trait at all. And `dyn Trait` is charged the union of every
   `impl` in the program (SPEC § 5.6), which a `[]` budget cannot pay.
2. `simplify` and `complicate` are mutation. `[mutate]` is a host
   effect. A pass here is a function from `[Int]` to `[Int]`.
3. One walk shrinks every strategy, dependent draws included.
4. A failure report carries a seed and a list, and they replay with no
   property, no generator and no machine.

**The cost, stated.** A shrink is minimal in the *choice sequence*, not
necessarily in the value. It works because every generator maps a
smaller choice to a simpler value; a `map` that inverts that order —
`n => 100 - n` over `int_range(0, 100)` — defeats it, and nothing can
detect that. hypothesis has the same cost and documents it the same
way; `strategy.map`'s own comment says so.

## Where the numbers come from

A `u64` seed and a splitmix64 step, **written in this package**. Three
alternatives, and why none of them:

| | why not |
| --- | --- |
| `[rand]` | a `core` package may not have it |
| a bound `Rand[e]` effect parameter | a trait bound cannot carry a type argument, so a bound generic over the value type cannot be spelled — and an impl the caller supplies is only reproducible if the caller made it so, which is the feature |
| the host fills a buffer and the core only reads it | hypothesis's literal shape; it works, and it makes the host guess how many integers a generation will need |

`tape.split_step` is `pub` because the whole reproducibility claim
rests on it: it is the compatibility surface, changing it changes every
seed's meaning, and it changes on a minor bump with a changelog line.

## The layer, and why

`core` — no effects, and randomness enters exactly once, in
`proptest-nv`, to *choose* a seed. Everything after that is a function
of the seed.

**No `@tier(embedded)` claim.** A device does not run a property-test
generator: the whole point of one is to run a hundred cases and shrink
a failure, which is a development activity. The audit's `core-embedded`
row passes as "makes no device claim", which is the honest reading and
not a dodge.

## The names, and the ones that were taken

| here | the obvious name | why not |
| --- | --- | --- |
| `PropStrategy<T>` | `Strategy` | the brief asked for a name that cannot collide, and flate-nv already prefixed its own (`FlateStrategy`) for the same reason |
| `Tape` | `Source`, `Buffer`, `Rng` | `Source` is a published enum, `Buffer` a standard-library struct, and `rng` is rand-nv's module name |
| `PropCheck`, `PropFailure<T>` | `Outcome`, `Failure` | both are published types already |
| `ShrinkPlan`, `ShrinkTry`, `ShrinkPass` | `Plan`, `Step`, `Pass` | `Plan` is a published enum, `Step` an orbit enum, and `pass` is a **reserved word** — it is the empty statement |
| `at_pass`, `by_pass` (fields) | `pass` | the same reserved word, which the compiler names with a hint |
| `PropHolds`, `ClassAscii`, `PassDeleteSpan` | `Holds`, `Ascii`, `DeleteSpan` | enum **variants** collide by bare name across the whole assembly |
| `bool_any`, `str_of`, `list_of` | `bool`, `str`, `vec` | `str` and `vec` are standard-library modules, and a function shadowing one makes every other call in the file ambiguous to read |

## The reference implementation

**hypothesis** for the model: strategies over a choice sequence,
`assume`, the shrink passes and their order, the overrun-is-a-discard
rule. **proptest** for the combinator vocabulary — `just`, `prop_map`,
`prop_filter`, `prop_flat_map`, `prop_oneof`, the size-range
collections — and for the insistence that a failure report carries the
seed. **QuickCheck** for `==>` as a first-class third answer rather
than a silent pass.

Deliberately left out, and where it went instead:

- **Running anything.** `proptest-nv`.
- **`Arbitrary`, and deriving a strategy from a type.** It needs an
  associated type and a blanket impl; the neighbouring filing is
  `no-blanket-impls-so-containers-need-one-fn-per-element-type`. A
  program writes its strategy, which is more typing and reads better in
  a failure.
- **Stateful / model-based testing.** A command sequence is a
  `list_of` a command strategy and a fold — a package that added a
  state machine on top of that is a separate row, not a section here.
- **Regex-driven string generation.** It wants a regex engine; the
  stdlib has one and it is `host`-shaped. `ClassOneOf` covers the
  alphabet case, which is most of it.
- **Coverage-guided generation.** It needs instrumentation, which is a
  toolchain feature and not a library one.

## Status

Every function is `todo()`. Two suites, and they are red for two
different reasons.

**`tests/tape_tests.nv`** reaches `not implemented:
proptest-core-nv.<fn>` on every assertion — the expected result until
the bodies land. Run it with `--isolate` for one verdict per test.

**`novo doc` reports 16 failing examples and `tests/strategy_tests.nv`
does not compile**, and that is one finding rather than an oversight. A
generic function whose signature applies a generic head to its type
parameter — `fn just<T>(v: T) -> PropStrategy<T>` — is not emitted for
a call from another module, so the whole combinator surface is
unreachable from anywhere but the module that declares it.

Four faces, all the same missing monomorphisation: an undefined symbol
at the LLVM verifier, an `E6000` internal compiler error on the call's
return type, a `ptr` stored into an `i64` slot when the result is
annotated, and an `E6000` on a field read off a `PropFailure<Int>`. The
same failure happens between two modules under `src/`, and inside the
scratch package `novo doc` builds for an example — which depends on
this one by path, exactly as a consumer's package does — so **no
consumer can call these functions either**. It is filed against the
LLVM backend as
`a-generic-fn-returning-a-generic-struct-is-not-emitted-for-a-cross-module-call`.

The package is **not** being reshaped around it. The alternative is a
monomorphic surface — `int_strategy`, `str_strategy`,
`int_list_strategy`, one function per element type — which is a
different library and a worse one. The examples are left as `novo`
blocks rather than fenced `novo ignore`, because they are code that
should compile and fencing them would hide the finding. When the defect
closes, both suites and every example go green with no edit.

| module | `pub` items | implemented |
| --- | --- | --- |
| `tape` | 1 type, 10 functions | no |
| `strategy` | 2 types, 20 functions | no |
| `shrink` | 3 types, 11 functions | no |
| `propcheck` | 2 types, 9 functions | no |
