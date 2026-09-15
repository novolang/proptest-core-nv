# Changelog

All notable changes to proptest-core-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

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
