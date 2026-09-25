# proptest-core-nv

A property-based test checks that a stated fact holds for many generated
inputs rather than for a few written ones. This package is the half of
such a tester that generates values and shrinks a failing one: a
**strategy** turns a recorded sequence of choices into a value, and
shrinking searches for a shorter sequence that still fails. The model is
[Hypothesis](https://hypothesis.works/articles/how-hypothesis-works/)'s,
and the combinator vocabulary is
[proptest](https://docs.rs/proptest/latest/proptest/)'s.
[proptest-nv](https://novo-lang.org/packages/proptest-nv) is the runner
built on this package, and
[fake-nv](https://novo-lang.org/packages/fake-nv) is fixture data built
on the same strategies.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What it is

A **property** is a fact about a program that should hold for every
input, such as "reversing a list twice gives the list back". A
property-based tester generates many inputs, checks the property on each
one, and when one fails it searches for a smaller input that fails the
same way. That search is called **shrinking**, and the small input it
finds is the **counterexample** a person reads.

A **strategy** is a rule for producing one value. Here it is a struct
holding a function, `PropStrategy<T>`, and every combinator is a
function that returns one. `int_range` produces an integer, `list_of`
produces a list of whatever another strategy produces, and `map` and
`flat_map` build a strategy out of a strategy.

A strategy does not draw from a random number generator. It draws from a
**tape**: a seeded source of integers that also records every integer it
gave. The recorded list is the **choice sequence**, or transcript, and a
generation is a function of it. A tape built from a seed draws as long as
it is asked to. A tape built from a transcript replays that transcript
and nothing else.

Shrinking is therefore arithmetic on a list of integers. A candidate is a
shorter or numerically smaller transcript, replayed through the same
strategy. One walk shrinks every strategy, including one whose later
draws depend on earlier ones, because the transcript is a flat list and
does not record which strategy read which part of it.

The seeded source is a splitmix64 step written in this package. There is
no randomness anywhere in it: a seed is an integer, and the same integer
produces the same values on any machine, in any process, with no clock
consulted. Randomness enters once, in proptest-nv, to choose the seed.

| Quantity | Value |
| --- | --- |
| Modules | 4 |
| Public functions | 50 |
| Public types | 8 |
| Shrink passes, tried in a fixed order | 6 |
| Answers a property may give | 3 |
| Effects any function in this package declares | none |

## Install

```
novo pkg add proptest-core-nv
```

## Example

```novo
use std.list
use propcheck
use shrink
use strategy
use tape

fn main() [io]
    // A strategy for a list of up to thirty-two integers, each in range.
    let ints = strategy.list_of(strategy.int_range(0 - 100, 100), 0, 32)

    // Turn one seed into one case. The tape that comes back holds the
    // transcript of every choice the strategy made.
    let (xs, played) = strategy.draw_with(ints, tape.from_seed(42))

    // The property being checked, as an ordinary boolean.
    let held = list.rev(list.rev(xs)) == xs

    // Turn the boolean into an answer. The text is used only if it fails.
    let answer = propcheck.holds_if(held, "reverse is not an involution")
    println("${list.len(xs)} values: ${propcheck.check_name(answer)}")

    // The line a failure report prints and a person pastes back.
    println(propcheck.replay_line(tape.seed_of(played), tape.choices_of(played)))

    // Minimising a failure starts here: a walk over the transcript.
    let plan = shrink.plan_from(tape.choices_of(played))
    println("${shrink.is_done(plan)}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: proptest-core-nv.<module>.<fn>` panic. The tests are
the specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `tape` | The seeded source of choices and the transcript it records: building one from a seed or from a transcript, the one primitive draw, the size-biased draw, and the splitmix64 step. |
| `strategy` | `PropStrategy<T>`, the byte classes a string strategy draws from, and the combinators: integers, floats, booleans, strings, lists, options, tuples, alternatives, `map`, `filter` and `flat_map`. |
| `shrink` | The six reduction passes over a transcript, the state of a shrink walk, the candidate it offers, and the ordering that decides what "smaller" means. |
| `propcheck` | The three answers a property may give, the minimised failure as a value, and the replay line that reproduces it. |

## How to choose an entry point

**`strategy.draw_with` is the only call that turns a strategy into a
value.** Everything else in `strategy` builds strategies. A runner calls
it once per case, and a program with no runner can call it directly.

**`tape.from_seed` generates; `tape.replay_of` reproduces.** Use the
first to make a case from a seed. Use the second to play a recorded
transcript back, which is what a shrink candidate and a saved failure
both are. `tape.replay_from` is the second one carrying the seed it came
from, for a report that wants both numbers.

**`strategy.from_draw` is the escape hatch.** A caller with a generator
this package does not offer wraps the drawing function in a
`PropStrategy<T>`, and it then composes with every combinator here.

**`shrink` is driven from outside.** `plan_from` starts a walk,
`next_try` offers a candidate, and the caller answers `keep` or
`reject`. Nothing in this package calls a property.

## The rules a user needs

1. **A smaller choice must mean a simpler value.** Every generator here
   obeys it: `int_range` maps choice 0 to `lo`, `bool_any` maps it to
   `false`, `option_of` maps it to `None`, `one_of` maps it to the first
   branch, and `list_of` draws its length before its elements. The
   shrinker rests on this rule and knows nothing else about your values.
2. **A `map` that reverses the order defeats the shrinker.** Mapping
   `n => 100 - n` over `int_range(0, 100)` makes a smaller choice a
   larger value, so the search walks away from the small counterexample.
   Nothing detects this. Use a function that preserves the order, or
   choose the shape with `flat_map` before the size.
3. **Put the simplest branch first in `one_of`, `weighted_of` and
   `sample_of`.** The branch is drawn as one choice, so shrinking walks
   a value towards the earlier branches. A recursive strategy whose first
   branch is not its base case does not bottom out.
4. **A shrink is minimal in the transcript, not necessarily in the
   value.** The walk finds the shortest, numerically smallest transcript
   that still fails. Hypothesis has the same limit and states it the same
   way.
5. **Every draw consumes a slot, even when there is no choice to make.**
   `tape.next` with a bound of 1 or less answers 0 and records it. Were
   the slot skipped, deleting a span from a transcript would change which
   slot every later draw reads.
6. **An overrun case must be discarded, not judged.** A tape from
   `replay_of` is sealed: when its transcript runs out it sets `overrun`
   and answers 0 to every further request. A value assembled from those
   zeros was never generated by the strategy, so a property failing on it
   has not failed on anything. `tape.is_overrun` is the signal, and
   `shrink.reject` is the answer.
7. **A property answers with a value, and there are three answers.**
   `PropHolds`, `PropBroke(why)` and `PropSkip`. `PropSkip` is
   Hypothesis's `assume` and QuickCheck's `==>`: the case did not meet
   the precondition and proves nothing either way. A runner counts skips
   separately, so a property that skips nine cases in ten is visible
   rather than looking like one that passed a thousand times.
8. **The shrink walk does not advance until it is answered.** Calling
   `shrink.next_try` twice without a `keep` or a `reject` offers the same
   candidate both times. That is what makes a walk resumable, and what
   lets a test assert one step of it.
9. **A transcript is not a seed.** `tape.replay_of` reports a seed of 0,
   because the transcript it holds is not the number that produced it.
   Use `replay_from` when both are known.
10. **`tape.split_step` fixes what every seed means.** It is the
    splitmix64 step, published so a caller can check this toolchain's
    arithmetic against the reference values. Changing it changes the
    value every seed produces, so it changes only on a minor version bump
    and the change is recorded in `CHANGELOG.md`.
11. **No function in this package performs input or output.** There is no
    randomness, no clock and no file. A failure is a value with fields,
    and rendering it is the runner's work.
12. **A trait cannot express this design in novo-lang, which is why a
    strategy is a struct.** A trait bound carries an effect argument and
    never a type argument (SPEC section 3.6), so a combinator generic
    over the value type cannot be written against a trait. A trait object
    is charged the effects of every implementation in the program
    (SPEC section 5.6), which a package that declares no effects cannot
    accept.

## What is not included

- **Running a property.** This package generates and shrinks. Choosing a
  seed, running the cases, driving the shrink walk and reporting the
  result are [proptest-nv](https://novo-lang.org/packages/proptest-nv).
- **Randomness.** A package that declares no effects cannot draw from the
  operating system. The seed is an integer a caller supplies, which is
  also what makes a failure reproducible from one printed number.
- **A generic helper of your own that forwards to these functions.** A
  function you declare outside this package, generic over the same type
  parameter, that calls `strategy.draw_with`, `strategy.just` or another
  generic function here, fails to build: the compiler emits a call to a
  symbol it never generated. Call these functions directly, or give your
  helper a concrete value type such as `PropStrategy<Int>`. The defect is
  recorded as
  `generic-forwarding-to-a-package-generic-skips-monomorphisation`, and
  when it closes the restriction goes away with no signature changing.
- **Deriving a strategy from a type.** An `Arbitrary` equivalent needs an
  associated type and a blanket implementation, neither of which the
  language has. A program writes its strategies, which is more typing and
  reads better in a failure report.
- **Stateful and model-based testing.** A command sequence is a `list_of`
  a command strategy and a fold over it. A state machine on top of that
  is a package of its own.
- **Strings generated from a regular expression.** It needs a regular
  expression engine, and the standard library's performs input and
  output. `ClassOneOf` covers generating from a fixed alphabet.
- **Coverage-guided generation.** It needs the compiler to instrument the
  program under test, which is a toolchain feature and not a library one.
- **Running on a microcontroller.** No such claim is made. Generating a
  hundred cases and shrinking a failure is a development activity, and a
  device does not do it.

## Related packages

- [proptest-nv](https://novo-lang.org/packages/proptest-nv) is the other
  half: it draws a seed, runs the cases, drives the shrink walk and
  reports the minimal case through `std.test`. Take that package to write
  property tests. Take this one to generate or shrink values with no
  runner in the program.
- [fake-nv](https://novo-lang.org/packages/fake-nv) is fixture data —
  names, addresses, dates — and each of its generators is a
  `PropStrategy<T>` from this package. A fixture and a property therefore
  draw from the same tape and compose with the same combinators.
- [rand-nv](https://novo-lang.org/packages/rand-nv) is general-purpose
  random numbers, including the operating system's entropy. Use it when
  you want randomness. This package exists to have none.
- `std.test` in the standard library is where a novo-lang test reports
  its verdict: `test.assert`, `test.fail` and `test.case`. This package
  does not call it. proptest-nv does, which is how a property failure
  reads like every other test failure.

## Tests

```bash
novo test tests/tape_tests.nv        # 15 tests: the tape, the shrink walk, the answers
novo test tests/strategy_tests.nv    # 12 tests: the combinators
```

The reference implementations are Hypothesis, for the choice-sequence
model, the shrink passes and their order, and the rule that an overrun is
a discard; proptest, for the combinator vocabulary and for the insistence
that a failure report carries its seed; and QuickCheck, for a skipped
precondition being a third answer rather than a silent pass.

No test draws a random number or reads a clock. Every case fixes a seed
or writes a transcript out by hand and asserts an exact answer, so none of
them can be flaky. The suite asserts that one seed always gives one
sequence of choices, that every draw is recorded even when the bound
leaves no choice, that a transcript replays exactly, that a sealed tape
overruns rather than inventing values, that the shrink order is shorter
first and then numerically smaller, that the six passes are a published
order, that the walk waits to be answered before it advances, that a
skipped case has not failed, and that a replay line parses back to the
seed and transcript it was printed from.

The tests compile today and fail at run, each on the
`not implemented: proptest-core-nv.<module>.<fn>` panic that is its body.
That is the expected state of an interface release. They turn green one at
a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `tape.from_seed`, `.replay_of`, `.replay_from` | no |
| `tape.next`, `.next_small`, `.split_step` | no |
| `tape.choices_of`, `.seed_of`, `.is_overrun`, `.remaining` | no |
| `strategy.draw_with`, `.from_draw`, `.just` | no |
| `strategy.int_range`, `.int_any`, `.float_range`, `.float_special`, `.bool_any` | no |
| `strategy.str_of`, `.list_of`, `.option_of` | no |
| `strategy.pair_of`, `.triple_of`, `.quad_of` | no |
| `strategy.one_of`, `.weighted_of`, `.sample_of` | no |
| `strategy.map`, `.filter`, `.flat_map` | no |
| `shrink.passes`, `.pass_name`, `.simpler_than` | no |
| `shrink.plan_from`, `.next_try`, `.keep`, `.reject` | no |
| `shrink.best_of`, `.shrinks_of`, `.tried_of`, `.is_done` | no |
| `propcheck.check_name`, `.broke`, `.reason_of` | no |
| `propcheck.holds_if`, `.assumed`, `.all_of` | no |
| `propcheck.failure`, `.replay_line`, `.parse_replay_line` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
