# lean-tests

A Claude Code skill that keeps agent-written test sets complete and minimal: one test per behavior, no tests that pin config values or copy the code, and guards only at trust boundaries. The skill itself is [SKILL.md](SKILL.md).

## Install

Link the checkout into your skills folder:

```bash
ln -s ~/Desktop/lean-tests ~/.claude/skills/lean-tests
```

Claude Code then offers the skill whenever a task plans, writes, reviews or consolidates tests, or adds a guard. To make a planner load it at fixed points, add a line such as this to your `CLAUDE.md`:

```text
Test plan rows, failing tests and the consolidation pass: load the lean-tests skill first.
```

## Sources

- Vladimir Khorikov, *Unit Testing Principles, Practices, and Patterns* (Manning, 2020): the four pillars of a good test, the code-type quadrant, mocks only for unmanaged dependencies.
- Kent Beck, "Test Desiderata" (2019): tests should be behavioral and structure-insensitive.
- Google Testing Blog, "Change-Detector Tests Considered Harmful" (2015).
- Thomas Ostrand and Marc Balcer, "The Category-Partition Method for Specifying and Generating Functional Tests," *Communications of the ACM*, 1988.
- Matthias Grochtmann and Klaus Grimm, "Classification Trees for Partition Testing," *Software Testing, Verification and Reliability*, 1993.
- RTCA DO-178C: modified condition/decision coverage (MC/DC).
- Mary Jean Harrold, Rajiv Gupta and Mary Lou Soffa, "A Methodology for Controlling the Size of a Test Suite," *ACM TOSEM*, 1993.
- Bertrand Meyer, *Object-Oriented Software Construction*, 2nd ed. (1997): Design by Contract and the non-redundancy principle.
- Alexis King, "Parse, Don't Validate" (2019).
