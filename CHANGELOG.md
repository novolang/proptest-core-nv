# Changelog

All notable changes to proptest-core-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-09-27

The first implementation of the interface published as 0.0.1.

### Added

- `tape`: splitmix64 seeded draws, sealed replays that overrun, and a
  size draw that is one choice, biased small on a fresh tape.
- `strategy`: every combinator, each keeping the rule that a smaller
  choice gives a simpler value.  A range at least `Int.max` wide is
  drawn in two choices.
- `shrink`: the six passes in their published order.  The delete pass
  also tries each deletion with the choice before it lowered by one,
  which is what removing a list's element does to its length.
- `propcheck`: the three answers, the failure value and the replay
  line `seed <n> choices <c>,<c>,...`.

### Changed

- `ShrinkPlan` has a field `round_start`, the accepted count when the
  current round of passes began; a round that accepted anything starts
  another.
- `strategy.one_of` and `strategy.sample_of` panic on an empty list,
  where the interface said they would answer a zero value, which a
  generic function cannot build.
- The toolchain floor is 0.14.0.  The bodies target novo 0.14.0 and
  carry no workaround for a compiler defect.

## 0.0.3 — 2026-09-25

The README example now reverses with `list.rev`.  Under novo 0.10.0
`list.reverse` reverses the list it is given in place, so the example
no longer compiled.  Had it compiled, reversing twice in place would have
compared the list with itself, and the property could not fail.
`list.rev` answers a new list.  No change to the interface.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-10

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `tape` — a seeded source that **records what it gave**, so a
  generation is a function of a list of integers and that list is kept.
  A `u64` seed and a splitmix64 step written here, because a `core`
  package may not touch `[rand]` and because reproducing a failure from
  one printed integer is the feature. `split_step` is `pub`: it is the
  compatibility surface, and changing it changes every seed's meaning.
- `strategy` — `PropStrategy<T>` as a closure in a struct, and the
  combinators over it: `just`, integer and float ranges, `bool_any`,
  strings from a byte class, `list_of`, `option_of`, the tuples,
  `one_of`, `weighted_of`, `map`, `filter`, `flat_map`, `sample_of`.
  A struct rather than a trait, because a trait bound in this language
  cannot carry a type argument and `dyn Trait` costs a `core` package
  its budget.
- `shrink` — hypothesis's internal shrinking: six named passes over the
  choice sequence, and a walk the **runner** drives, because calling a
  property would cost whatever the property costs.
- `propcheck` — three answers rather than two, so a precondition that
  skips a case is not counted as a pass, and a failure that is a value
  rather than printed text.

### Known

- **`tests/strategy_tests.nv` does not compile.** A generic function
  whose signature applies a generic head to its type parameter is not
  emitted for a call from another module, so the combinator surface is
  unreachable from anywhere but `strategy.nv` itself — for a consumer
  as well as for a test. The filing is
  `a-generic-fn-returning-a-generic-struct-is-not-emitted-for-a-cross-module-call`.
  The package is not being reshaped around it; when it closes, the
  suite reaches `not implemented` with no edit.
- No `@tier(embedded)` claim, and none is intended: a device does not
  run a property-test generator.
