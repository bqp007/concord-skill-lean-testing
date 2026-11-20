---
name: concord-skill-lean-testing
description: Plan, write, review and consolidate tests so the test set is complete and minimal — every behavior the change decides has exactly one test, no test pins config values or copies the code, and no defensive guard adds branches that exist only to be tested. Use this skill whenever you write a test plan, add or change tests, add a regression test for a bug fix, consolidate tests after a change, review a test diff, decide whether something needs a test at all, or are about to add a guard, an assert or a validation check — even when the request only says "add tests" or "make it robust".
---

# Lean testing

A test earns its place when it catches a fault that no other test catches and passes on every change the spec allows. A test set is finished when it is complete (every behavior and every changed branch is accounted for) and minimal (each test guards a distinct behavior and touches only the public interface). Extra tests are not free: each one is code to maintain, and a test that breaks on legitimate changes slows every later change without protecting anything.

Agents drift toward too many tests for three reasons. Tests written right after reading the implementation assert what the code does, not what the spec requires. More tests feel safer, and nothing says when to stop. Tests are rarely deleted. This skill gives a rule for what a test may assert, a procedure, and a stopping check.

## The core rule: the expected value must outrank the code

A failing test tells you something only when its expected value comes from a source with more authority than the code under test. Then the failure means the code is wrong.

Sources that outrank the code:
- the spec, the acceptance criteria, or a documented requirement;
- a contract that another module, repository or team consumes: an API, a schema, a wire or file format;
- a reference implementation the code must match;
- a mathematical law or invariant: round trip, idempotence, ordering, monotonicity;
- a golden file that the spec owner reviewed and approved.

Sources with the same authority as the code:
- config values: thresholds, lists, prompts, model names, version pins, flags;
- the code's own output, recorded without review (a snapshot);
- constants copied from the code;
- the algorithm re-implemented inside the test.

When a same-authority test fails, nobody can tell which side is wrong, so the fix is to make the two agree again. Such a test catches no fault, and it fails on every legitimate change. Rewrite it against a higher source or delete it.

A good test set works like a filter on changes. It rejects every change that breaks a promise. It accepts every change the spec allows: refactors, config retunes, performance work, added features, renames and log text. A test that rejects an allowed change is as much a defect as a test that accepts a broken one.

## 1. Classify what the change touches

Decide what kind of thing each touched part is before you write a test. The kind decides whether it gets one.

| Kind | How to recognize it | What it gets |
|---|---|---|
| Logic | rules, decisions, transforms and calculations on inputs | unit tests through the public interface, one per behavior class |
| Contract | something outside the unit reads it: an API, a schema, a file or wire format | one contract test per boundary that checks values, not only shape, usually against a golden file the spec owner approved; a schema check alone accepts a wrong value of the right type |
| Config | a value someone tunes (see below) | no tests on its values; unit tests inject their own config; the loader checks the rules of the shipped files at load, and one release check runs it on them |
| Quality on real data | the result only means something with production data, models or config | an evaluation with a metric and a threshold the owner sets, outside the unit suite |
| Glue | wiring, controllers, composition: many collaborators, little logic | one integration path that exercises it |
| Trivial or covered elsewhere | getters, data classes, properties the compiler or type checker proves, third-party behavior | no tests |

### Config or contract?

A value is config when the spec names the parameter but not its value, when it changes for tuning rather than for correctness, and when it lives in a config file, a constants module, a bundle or an environment variable. A test that asserts a config value only repeats the config file. Whoever retunes the value must also edit the test, so the test catches nothing and adds a maintenance step to every retune.

A value is a contract when the spec fixes the exact value: a legal retention period, a protocol constant, a published limit. A contract value gets one test.

Shipped config still obeys rules that outrank its values: required keys, types, ranges, and references between files, such as every route naming a label that the label file defines. Put those rules in the loader, so that production fails fast too, and add one release check that runs the loader on the shipped files. That check reads production config but asserts no tuned value, so a retune that keeps the rules passes it.

Test the logic around a config value with values the test chooses:

| Instead of | Write |
|---|---|
| `assert cfg.threshold == 0.40` | Inject `threshold = T`. Assert that a score of `T` is flagged and `T - ε` is not. The test survives any retune and still catches `>` written for `>=`. |
| `assert "bad.example" in blocklist` | Inject the list `[a]`. Assert that `a` is blocked and `b` is not. Which entries belong on the list is a curation review, not a test. |
| `assert prompt == "You are..."` | Inject a template. Assert that the item lands in its slot. Prompt quality is an evaluation. |
| `assert MODEL_VERSION == "2.3.1"` | If the code refuses a model with another major version, inject two versions and assert the refusal. |
| `score(0.41)`, where `score` takes its default threshold from config | Pass the threshold explicitly. The default couples the test to config without showing it. |

## 2. List the behavior classes

For each spec clause the change touches, split the inputs into classes that must produce the same kind of result. Add a boundary case where a class has an edge. Each class is a leaf of the behavior tree, and each leaf gets one test. A layer that transforms a value on its way through, such as a serializer, a mapper or a parser, adds leaves of its own, because it can break a value that the layer below computed correctly. In a table-driven test, each row counts as one test. Where a law covers a whole class (round trip, ordering), one property test can stand for that class.

For a decision with two or more conditions joined by and/or, use modified condition/decision coverage (MC/DC): for each condition, a pair of cases that differ in that condition alone and give different results. Pairs share cases, so N conditions take at least N+1 cases and at most 2N; find the pairs before you set the budget. Branch coverage alone can pass while a condition never decides anything; MC/DC is the cheapest set that shows each one does. When a condition appears twice, as A does in `(A && B) || (A && C)`, no pair can change one appearance alone. Simplify the decision first (`A && (B || C)`). Where it cannot be simplified, let the other appearance change too while another condition masks it (masking MC/DC).

The number of leaves sets the test budget. The number of changed lines does not.

## 3. Reuse before you add

For each leaf, find the existing test that covers it, then pick one action:
- KEEP: an existing test covers the leaf and stays correct.
- EXTEND: add a row or an assert to the existing test.
- REPLACE: the existing test is wrong for the new behavior.
- MERGE: two tests guard the same leaf; keep one.
- DELETE: the change removes the leaf, or the test breaks the core rule.
- ADD: only for a leaf that no test covers.

A bug fix adds a test only when the bug's leaf had no test. When a test covered the leaf and let the bug through, fix that test instead.

## 4. Write each test

- Write the test before the implementation, from the clause. Take the expected value from the spec, not from running the code or reading a constant.
- Call only the public interface: the function, endpoint or command that real callers use. Do not call private symbols, and do not extract a helper only so that a test can call it. Tests on private code break on every refactor.
- Build the config a unit test needs inside the test. Only the release check from step 1 reads the shipped config files.
- Mock only out-of-process dependencies that the code does not own, such as network services and third-party APIs. Inject the clock and the random source. Use real instances of everything else; a mock of your own module freezes its internal calls.
- Name the test after its leaf, and be able to say in one line which fault it catches. If another test catches the same fault, merge the two.
- Test each leaf at the lowest interface that is a contract: one that callers outside the unit use and that a refactor inside the unit leaves alone. That is usually the cheapest layer that shows the fault. Do not test the same fault again at another layer.
- Assert only the result of the behavior the test names: no asserts on the setup and no asserts inside test helpers.
- Keep wall-clock timing out of unit tests. A benchmark measures speed.

## 5. Add guards only at trust boundaries

A routine does not re-check its own preconditions; its callers own them. Parse and validate input once, where it enters the system: network, files, user input, deserialization, config load. Inside that boundary, trust the types and the callers.

State that something else can change between the check and the use is input too: a balance that two requests spend at once, a row, a file, a remote resource. Both requests can pass validation and still overspend. Enforce that invariant where the state changes, in one atomic step: a conditional update, a constraint, a lock or a compare-and-swap. That check is a trust boundary, not a defensive copy.

Every guard adds a branch, and every branch needs a test or a reason. A guard for a state that cannot occur doubles the cost and protects nothing. Before you add a guard, check two things: that a real input can reach this line in that state, and that the code does not already enforce it. Where you can, choose a type that cannot hold the bad state instead of a check that rejects it.

## 6. Check completeness and minimality, then stop

After the implementation passes, check these six points. When all six hold, stop writing tests.

1. Every leaf from step 2 has exactly one test.
2. Every branch that the diff adds or changes is reached by a test, or has one of these reasons: glue covered by an integration path, third-party behavior, logging or metrics only, guaranteed by the type system. Run the repository's branch-coverage tool on the diff to find unreached branches. Record each reason in the test plan, not as a coverage pragma in production code. "Unreachable" is not a reason: delete that code.
3. A changed branch that no clause asks for is an undecided behavior. Ask the owner to decide it, or delete it. A test would make the undecided behavior permanent.
4. Every test maps to a leaf. Delete a test that maps to none.
5. No unit test reads production config, and no test asserts a tuned config value or reaches a private symbol.
6. Every expected value comes from a source that outranks the code.

A coverage percentage is not the goal; a percentage target rewards extra tests. The goal is that every branch is accounted for.

## 7. Consolidate at the end of the task

Go over every test that the task touched. Merge tests that guard the same leaf. Delete tests whose leaf is gone or that fail point 5 or 6 of step 6. Keep the test lines within the budget from step 2.

## Reviewing tests

A finding on a test needs evidence in one of two forms:
- a wrong implementation that the test still accepts (the test cannot fail), or
- an allowed change that the test rejects: a retuned config value, or a refactor that keeps behavior (the test pins something it should not).

Without one of these, the finding is a question, not a defect.

## Limits

- Completeness holds only relative to the leaves you listed and the branches you measured. Whether the leaves match the real requirement is for the spec and its owner to settle.
- Coverage shows that a branch ran, not that a test checked its result. Mutation testing on the diff shows the second, but it costs much more; run it only when asked.
