---
layout: post
title: "Code Health as Context: Making Agentic Coding Safer and More Efficient"
date: 2026-11-08 01:00:00 +0100
categories: [software, ai]
tags: [agentic-coding, code-health, maintainability, llm, go]
---

# Code Health as Context: Making Agentic Coding Safer and More Efficient

AI coding agents are very good at producing a plausible patch quickly. They are
less good at knowing whether the patch belongs in a codebase that has become
hard to understand, risky to change, or expensive to explain.

That distinction matters. “The code compiles” is not the same as “the next
change will be safe.” A feature can pass its tests while adding another large
function, another dependency edge, or another copy of an existing block. The
cost is paid later: by the next developer, the next reviewer, and the next
agent that must reconstruct the system from a larger context.

I built [sghaida/skills](https://github.com/sghaida/skills) to make that
condition visible to an agent. Its first skill, [`chnq`](https://github.com/sghaida/skills),
is a dependency-free Go analyzer that produces a deterministic, explainable
structural maintainability score from 0 to 10. The goal is not to replace
tests, code review, or domain judgment. The goal is to give agentic coding a
small, repeatable feedback loop:

1. measure the repository before changing it;
2. make the smallest change that satisfies the request;
3. measure it again;
4. do not accept a regression through the configured quality gate.

This is a modest mechanism, but it addresses a large problem: agents need
guardrails that are cheap enough to run on every task.

## The problem is not only generating code

An agentic coding task usually has two different costs:

* **generation cost**: producing a patch;
* **orientation and verification cost**: finding the right files, building a
  correct mental model, checking assumptions, and validating the result.

The second cost is easy to underestimate. A repository with long functions,
high branching, repeated logic, and many internal dependencies requires more
searching and more explanation before a safe edit can be made. The agent may
need to read more files, ask more questions, retry more often, and review a
larger diff.

In an LLM application, that work is represented partly as context. Context is
made of tokens, and tokens are not an abstract concern: providers expose input
and output token usage as a billable and operational metric. More context can
also mean more latency, and a context window is finite. Reducing irrelevant
context is therefore useful, but “fewer tokens” is not a sufficient definition
of a good engineering task. A short, incorrect patch is not efficient.

The useful target is:

> **minimum sufficient context for a correct, maintainable change.**

Code health helps establish what “sufficient” means. It gives the agent a
current baseline and identifies hotspots instead of asking it to treat every
file as equally risky.

## What `chnq` measures

The analyzer is intentionally narrow. It currently targets Go repositories
and never executes the analyzed repository. It reports four structural
signals:

| Dimension | Weight | Signal |
| --- | ---: | --- |
| Complexity | 40% | Branches, boolean conditions, nesting, and control-flow constructs |
| Structure | 25% | Oversized functions, deep nesting, and oversized files |
| Duplication | 20% | Repeated contiguous 35-token blocks across functions |
| Coupling | 15% | Go-module-local package dependencies |

The overall score is a weighted average on a 0.0–10.0 scale. Scores are also
reported for packages and files, and the analyzer identifies low-scoring
hotspots. That last part is important for an agent: a repository score can
look acceptable while one file is a poor place to add another feature.

This is a heuristic, not a universal definition of maintainability. It does
not measure semantic correctness, test quality, security, cohesion, Git churn,
architecture violations, or semantic clones. The repository README documents
those boundaries explicitly. A score is a signal for prioritization, not a
certification.

### A concrete policy

The skill supports a checked-in `.go-code-health.json` policy:

```json
{
  "version": 1,
  "duplication_engine": "internal",
  "minimum_score": 7.0,
  "minimum_dimensions": {
    "complexity": 6.0,
    "structure": 7.0,
    "duplication": 8.0
  },
  "include_tests": false
}
```

An agent can run a baseline scan before coding and then enforce the same
threshold after the patch:

```sh
./chnq/scripts/analyze.sh /path/to/repository --format json
./chnq/scripts/analyze.sh /path/to/repository --fail-under 7
```

If the configured policy already fails, remediation takes priority over an
unrelated feature. This is deliberately stricter than producing a
success-shaped answer and leaving the repository worse than it was.

## Why this should help an agent

The mechanism is not “ask the model to write cleaner code.” It turns a vague
instruction into an observable invariant.

Without a health gate, a typical request can look like this:

```text
Find the right code, implement the feature, and tell me when it is done.
```

With a health gate, the task has an additional contract:

```text
Understand the baseline, implement the feature, and preserve the configured
health score and dimensions.
```

That changes the agent's decision-making in several useful ways:

* **It narrows exploration.** Hotspots and package boundaries are evidence
  about where to look first.
* **It makes refactoring accountable.** A refactor is not “cleaner” because it
  sounds cleaner; it should improve a measured signal without changing
  behavior.
* **It prevents silent deterioration.** A passing build cannot hide a new
  complexity or duplication regression.
* **It creates a compact handoff.** A score, dimension breakdown, and hotspot
  list are cheaper to carry between agent turns than an unstructured tour of
  the entire repository.
* **It supports incremental work.** The agent can fix one hotspot, remeasure,
  and continue rather than attempting an unbounded rewrite.

The score is especially useful as a *routing signal*. A healthy, well-bounded
package may need a focused implementation. A low-scoring hotspot may require
tests, decomposition, or a smaller change first. The tool does not make that
decision automatically; it supplies evidence for the decision.

## What public research says about speed

There is no single productivity number that applies to all agentic coding.
Published results vary with task type, developer experience, repository
familiarity, tool, and how “productivity” is measured.

In a controlled experiment, [Demirer et al.,
*The Impact of AI on Developer Productivity*](https://arxiv.org/abs/2302.06590)
reported that developers with access to GitHub Copilot completed an HTTP-server
task **55.8% faster** than the control group. That is a meaningful result, but
it is one task and one experimental setting; it does not prove that every
repository or every feature becomes 55.8% faster.

The counter-evidence is equally important. The [METR randomized controlled
trial, *Measuring the Impact of Early-2025 AI on Experienced Open-Source
Developer Productivity*](https://arxiv.org/abs/2507.09089), studied 16
experienced developers completing 246 tasks in mature projects. Developers
expected AI to reduce completion time by 24%, and estimated a 20% reduction
afterward, but the measured result was a **19% increase in completion time**
when AI tools were allowed. The authors discuss project familiarity, quality
standards, and the cost of reviewing generated work as possible contributors.

These results are not contradictory once the task is described accurately.
AI can accelerate a bounded, well-specified implementation while slowing work
that requires deep repository understanding and careful verification. That is
exactly the environment where structural code health can matter: it reduces
the amount of accidental complexity an agent must reconstruct, while still
requiring the agent to prove that its change is safe.

The practical conclusion is not “AI is fast” or “AI is slow.” It is:

> Measure the complete loop, including orientation, implementation, review,
> rework, tests, and post-change quality.

## Readability and maintainability are delivery properties

Readability is often treated as a preference, but maintainability has an
observable economic effect. SonarSource's [technical-debt
guidance](https://www.sonarsource.com/resources/library/technical-debt/)
describes debt as a cost that compounds through slower changes, more defects,
and more difficult onboarding. Its recommended direction is to measure,
prioritize, and apply quality gates continuously rather than waiting for a
large cleanup project.

`chnq` follows that operational idea while keeping its scope explicit. It
does not claim to measure all readability. Instead, it measures structural
properties that often make code harder to read: deep nesting, excessive
branching, very long functions and files, duplicated blocks, and local
coupling.

This distinction also prevents a common failure mode in AI-assisted
development: optimizing for a metric while ignoring the software. A lower
score is not automatically a reason to rewrite working code. The policy
should be paired with tests, review, and domain constraints. The right
question is whether the change leaves the system easier or at least no harder
to understand.

## Where tokens fit—and where the evidence stops

It is tempting to claim that better code health directly produces a specific
token reduction. The public evidence does not justify a universal percentage.
Neither `chnq` nor the research cited above claims such a relationship.

There is, however, a clear engineering mechanism:

1. A tangled repository increases the search and explanation required before a
   safe edit.
2. Search results, file excerpts, plans, generated patches, and retries become
   model input and output.
3. A health baseline and hotspot report can reduce irrelevant exploration and
   focus verification.
4. Fewer irrelevant tokens can reduce latency and cost, provided correctness
   is preserved.

This is a hypothesis to measure per project, not a benchmark to invent.
Anthropic's public [long-context prompting
guidance](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices#long-context-prompting)
also recommends structuring long inputs and placing documents before the query.
That is useful context-engineering advice, but it is not evidence that a
particular maintainability score saves a fixed number of tokens.

Code models themselves show why context design matters. The [Code Llama
paper](https://arxiv.org/abs/2308.12950) reports training on 16k-token
sequences, improvements on inputs up to 100k tokens, and benchmark results
such as up to 67% on HumanEval for its reported variants. Larger context
capacity makes more input possible; it does not make all input useful. A
smaller, relevant context can still be preferable to a large dump of a
repository.

## A measurement plan for teams

Teams adopting this approach should publish their own baseline instead of
repeating a vendor claim. For each comparable task, record:

* input and output tokens;
* wall-clock time;
* number of tool calls and retries;
* time to first useful patch;
* review and rework time;
* tests and static-analysis results;
* pre-change and post-change health scores;
* defect or rollback rate after the change.

Compare at least two conditions:

1. agentic coding with the health scan and hotspot report;
2. the existing workflow without that signal.

Keep task difficulty and repository area as similar as possible. Report
medians and distributions, not just the best example. A useful outcome might
be “the health-aware workflow used 12% fewer input tokens with no increase in
rework,” not “the skill makes agents 12% faster.” The former is a measurable
observation; the latter is an unsupported generalization.

The most important acceptance criteria are still behavioral:

* tests pass;
* the feature meets its requirements;
* security and type checks pass where applicable;
* the health policy is met;
* the resulting diff is understandable to a human reviewer.

## A small feedback loop beats a large promise

The purpose of [sghaida/skills](https://github.com/sghaida/skills) is to make
that loop reusable. `chnq` is intentionally dependency-free, deterministic,
and explainable so it can run before and after an agent task. It is not a
replacement for a full quality platform, and it is not proof that a codebase
is correct.

It is a way to give an agent a memory of the repository's structural health:

```text
baseline -> focused change -> tests -> health scan -> review
```

When the score falls, the agent has an explicit reason to stop, explain the
regression, and improve the design. When the score holds and the tests pass,
the team has stronger evidence that speed did not come from borrowing more
technical debt.

That is the balance I want from agentic coding: not maximum generated code,
but safe, clean changes delivered with the smallest context that is sufficient
to understand and verify them.

## References

* [sghaida/skills](https://github.com/sghaida/skills)
* [Demirer et al., The Impact of AI on Developer Productivity](https://arxiv.org/abs/2302.06590)
* [Rush et al., Measuring the Impact of Early-2025 AI on Experienced Open-Source Developer Productivity](https://arxiv.org/abs/2507.09089)
* [SonarSource, Technical Debt FAQs](https://www.sonarsource.com/resources/library/technical-debt/)
* [Anthropic, Prompting best practices](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices#long-context-prompting)
* [Rozière et al., Code Llama: Open Foundation Models for Code](https://arxiv.org/abs/2308.12950)
