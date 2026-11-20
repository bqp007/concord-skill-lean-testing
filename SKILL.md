---
name: concord-skill-lean-testing
description: Plan, write, review and consolidate tests so the test set is complete and minimal — every required behavior is covered, every test catches a failure the rest of the suite would miss, no test pins config values or copies the code, new tests are placed against the existing suite instead of piled on top of it, and no defensive guard adds branches that exist only to be tested. Use this skill whenever you write a test plan, add or change tests, add a regression test for a bug fix, find that a new test duplicates or contradicts an existing one, decide what to do with an existing test that a change turned red, consolidate tests after a change, review a test diff, decide whether something needs a test at all, or are about to add a guard, an assert or a validation check — even when the request only says "add tests" or "make it robust".
---

# Lean testing

A test earns its place when it catches a failure that no other test catches and passes on every change the spec allows. Extra tests are not free: each one is code to maintain, and a test that breaks on legitimate changes slows every later change without protecting anything.

Agents drift toward too many tests for three reasons. Tests written right after reading the implementation assert what the code does, not what the spec requires. More tests feel safer, and nothing says when to stop. Tests are rarely deleted.

This skill states the principles and what a finished test set must show. It does not prescribe a technique or a count. How you find the behaviors, and how many cases they take, is your judgment.

## Principles

### The expected value must outrank the code

A failing test signals a mismatch between the code and an expectation. The failure helps only when a source independent of the code can settle which side is wrong.

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

When a same-authority test fails, nobody can tell which side is wrong, so the fix is to make the two agree again. Such a test catches no failure, and it fails on every legitimate change. An unreviewed snapshot is a change detector, not a test, until someone establishes the behavior it should show. A reviewed golden file or a fixture can also be stale, so when a test fails, go back to the source to decide whether the code or the test changes.

### The test set is a filter on changes

A good test set rejects every change that breaks a promise. It accepts every change the spec allows: refactors, config retunes, performance work, added features, renames and log text. A test that rejects an allowed change is as much a defect as a test that accepts a broken one.

### Config is not behavior

A value is config when the spec names the parameter but not its value, and the value changes for tuning rather than for correctness. A test that asserts a config value only repeats the config file: whoever retunes the value must also edit the test, so the test catches nothing. A value is a contract when the spec fixes it exactly, such as a legal retention period or a protocol constant, and then a test may assert it.

Test the logic around a config value with values the test chooses. For "flag a score at or above the threshold", inject a threshold `T` and assert that a score just below `T` is not flagged, that `T` is flagged, and that a score above `T` is flagged. The test survives any retune. The case at `T` catches `>` written for `>=`, the case above `T` catches `==`, and the case below `T` catches a rule that flags everything. Two inputs with the same expected result both earn their place when they catch different failures.

Config still needs checks, and none of them asserts a tuned value:
- The shipped files obey rules that outrank their values: required keys, types, ranges, and references between files. Enforce those rules in the production loader, so that production fails fast too, and run that loader on the final shipped artifacts.
- Which profile is loaded and which override wins is behavior. When it changes, test it through the normal entry point, with fixture values you can tell apart.
- A result that only means something with production data, models or config belongs in an evaluation, with a metric and a threshold the owner sets. Release checks and evaluations may read shipped config when it is part of what they verify.

### Test through contracts

Test behavior through an interface that real callers use and that a refactor inside the unit leaves alone. A test that reaches a private symbol, calls a helper extracted only for the test, or mocks one of your own modules is tied to how the code is built, not to what it promises, so it breaks on refactors that keep behavior. Which layer to test at is your call: pick the one where the promise is stated and a failure shows clearly.

### Guards belong at trust boundaries

A routine does not re-check its own preconditions; its callers own them. Validate input once, where it enters the system, and trust the types and the callers inside that boundary. Every guard adds a branch, and every branch needs a test or a reason, so a guard for a state that cannot occur costs twice and protects nothing. Where you can, choose a type that cannot hold the bad state.

State that something else can change between the check and the use is input too: a balance that two requests spend at once, a row, a file, a remote resource. Enforce that invariant where the state changes, in one atomic step. That check is a trust boundary, not a defensive copy.

### The suite outlives the task

The tests you write join a suite that other tasks wrote and later tasks will change. Each test is a standing claim: this behavior, this expected result, from this source. A task changes that set of claims, so place each new case against the tests that are already there:

- It catches a failure that no existing test catches: add it beside the tests of that behavior.
- An existing test already catches what it catches: do not add it.
- It catches what an existing test catches, and more: replace the weaker test.
- It covers a new input class of a behavior that already has a test: extend that test.

A new expectation that contradicts an existing test is a conflict between requirements, not between tests. An existing test that turns red is evidence first. The source decides among three cases. The requirement changed, so the old expectation is replaced and the change names the requirement. The old test pinned an incidental detail, so the test is corrected or deleted. Or the new code is wrong, so the code changes. When two sources of equal rank disagree, ask the owner. Do not edit an expectation until it passes without saying which case applies.

A test whose behavior the change removed still passes, so nothing flags it. Delete it in the same change.

Maintain the tests of the behaviors the task touches and leave the rest alone; a suite-wide cleanup is a task of its own. Do not remove a test because another test runs the same lines: equal coverage does not mean that the same failures are caught.

Placement works only when a test says what it guards. Name each test for its behavior, keep it with the other tests of that behavior and not under a ticket or bug number, and make its source findable. A test of unknown origin is a change detector until someone establishes what it should show.

## What a finished test set shows

**Complete**

- Every distinct behavior that the change must have, including its failure modes, is covered by at least one assertion that fails when the behavior breaks. A case may cover several behaviors. Behaviors come from the requirement, not from the lines changed.
- No path through the changed code is left unaccounted for. Each one is exercised, or you can say where it runs (another platform, a deployment variant), which invariant prevents it, or what prevents verification. A required behavior that could not be verified stays open: report the gap, and do not count it as covered. Delete code only on evidence that it is dead across the supported configurations. A coverage percentage is not a goal.
- A changed path that introduces an observable behavior no requirement asks for is an undecided behavior. Ask the owner to decide it, or remove it; a test would make it permanent. A path that keeps behavior, such as an equivalent fast path, needs no decision.

**Minimal**

- Every case catches a failure that the rest of the suite would miss, and you can say which one. Merge or remove a case when the remaining suite preserves its coverage.
- Existing tests come first. Keep, extend or correct the test that already covers a behavior, and add or extend a case only when it catches a failure that the suite misses. When a test covered the behavior and let a bug through, fix that test.
- No test asserts a tuned config value, repeats the algorithm, or is tied to incidental detail such as private symbols, its own setup or wall-clock time.
- No guard exists for a state that cannot occur.

**Consistent with the suite**

- Every new case was placed against the existing tests of its behavior: added beside them, merged into one of them, or dropped as a duplicate.
- Every expectation that the change edited or deleted names the requirement that changed, or the incidental detail the old test pinned.
- No test remains for a behavior the change removed.
- Tests of behaviors the task did not touch are unchanged.

When all three hold, stop writing tests. At the end of a task, go over the tests it touched once more against these points.

## Reviewing tests

A finding on a test names the affected behavior and shows evidence, such as:
- a wrong implementation that the test still accepts (the test cannot fail);
- an allowed change that the test rejects, such as a retuned config value or a refactor that keeps behavior (the test pins something it should not);
- a required case that is skipped, never collected, or nondeterministic.

Without evidence, the finding is a question, not a defect.

## Limits

- Completeness holds only relative to the behaviors you listed and the paths you examined. Whether the behaviors match the real requirement is for the spec and its owner to settle.
- Coverage shows that a branch ran, not that a test checked its result. Mutation testing on the diff shows the second, but it costs much more; weigh that cost against what the project requires.
- Redundancy across a whole suite cannot be settled by reading the tests or by comparing coverage. It takes a record of which test catches which fault, and whether to pay for that is a decision for the project.
