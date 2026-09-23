---
layout: post
title: "When accuracy of 38% and 66% is both possible and revealing"
---

# TLDR
{: .no_toc}

In my [previous post]({% post_url 2026-06-18-evals-for-ai-agents %}), a text-to-SQL agent scored 38% on execution accuracy (EX) and 66% on the LLM judge. I went looking into the gap.

* **EX and LLM judge gap**: the two graders are testing different notions of correctness.
* **EX can both miss real SQL bugs and reject valid answers.** The useful move is to inspect those disagreement buckets instead of averaging them away.
* **Lesson for agent evals:** deterministic graders verify observable behavior; semantic graders test task success. Their disagreements are often the best debugging signal.


# Contents
{: .no_toc}

* TOC
{:toc}

In my [previous post]({% post_url 2026-06-18-evals-for-ai-agents %}), I built an eval for a text-to-SQL agent. One result stood out: strict execution accuracy (EX) was 38%, while the LLM-judge-adjusted score was 66%.

![Breakdown from EX to judge verdicts](assets/ex-vs-llm-judge/ex-llm-example-counts.png)

That is a large gap. So I went through the disagreement cases to understand what each grader was actually measuring. The lesson was not "EX is too strict" or "LLM judges are better". It was this:

**When deterministic and semantic graders disagree, the disagreement is often the most interesting part of the eval.**

## A quick recap of the agent and graders

My agent is simple enough:

```text
user asks free-form question
        ↓
inspect schema / sample data
        ↓
write SQL
        ↓
run SQL
        ↓
inspect results and optionally revise
        ↓
write a free-form answer
```

For every run, the harness saves the trace:

* user's question
* tools called
* SQL queries
* returned rows
* final answer
* errors, if any

That lets us grade more than just the final prose.

The two graders relevant here are **execution accuracy (EX)** and an **LLM judge**.

### 1. Execution accuracy

EX asks:

> Did the agent SQL return the same result as the gold SQL?

That immediately raises a less trivial question: what does "the same" mean?

You have several plausible choices:

* **Compare values by position.** This is what my eval does. Row order and column names are ignored, but values must stay in the same output positions. Extra columns therefore fail on non-empty results.
Example:
```sql
SELECT annual_tribute, province  -- gold SQL
SELECT province, annual_tribute  -- agent - DIFFERENT
```

* **Align columns by name.** This handles reordered columns, but aliases and duplicate names quickly make things ambiguous.
Example:
```sql
SELECT province, annual_tribute              -- gold SQL
SELECT province_name, annual_tribute_amount  -- agent - DIFFERENT
```

* **Compare only the fields needed to answer the question.** This can tolerate harmless extra columns, but now someone has to decide which fields matter. At that point your "deterministic" grader has acquired opinions.
Example:
```sql
SELECT annual_tribute, province            -- gold SQL
SELECT province, annual_tribute, governor  -- agent - DIFFERENT
```

even if `governor` is harmless extra information.

The alternative (deciding that only `province` and `annual_tribute` matter) requires understanding the question, which is exactly where the second grader comes in.

### 2. LLM judge

The LLM judge takes a different path depending on EX. If EX **fails**, it asks whether the agent SQL is nevertheless an acceptable answer to the question. If EX **passes**, it checks for false positives: cases where the result happens to match the gold result, but the SQL logic is still wrong.

## What EX actually tells us

EX is wonderfully simple: run the agent SQL, run the gold SQL, compare the results.

```text
agent SQL on this database == gold SQL on this database?
```

It is deterministic and cheap.

But an EX pass proves less than we often assume. What we usually care about is whether the two queries would **keep behaving the same way as the database changes in ways the application allows**.

If a column is nullable, eventually someone may put a null in it. If names are not unique, two Romans may eventually share one. If a province is allowed to exist without a governor, eventually one may. EX checks one snapshot of that world, and sometimes that snapshot simply does not contain the row that would expose the bug.

## The disagreement matrix

Once you distinguish "same result here" from "same behavior more generally", the EX-versus-judge gap becomes easier to reason about. Looking at both graders gives four buckets:

|                   | EX pass          | EX fail           |
| ----------------- | ---------------- | ----------------- |
| **Judge accepts** | expected success | valid alternative |
| **Judge rejects** | latent SQL bug   | expected failure  |

The diagonal is useful but boring (in a good way). It's the off-diagonal cells are where the eval starts teaching you things:
* **EX pass + judge reject:** the current data may be hiding a semantic difference.
* **EX fail + judge accept:** the deterministic comparison, gold answer, or output contract may be stricter than the task itself.

Those two cases explain a lot of the gap.

## EX pass, SQL wrong: the missing counterexample

Start with the stranger case: *how can SQL be wrong if it returned exactly the right rows?*

Imagine a Roman tax database and a question asking for total tribute by region. The candidate query silently excludes provinces whose region is unknown. The reference query does not.

Today, every province happens to have a region. Both return identical results. EX passes. Tomorrow, someone adds *Provincia Incognita*, whose region is legitimately unknown. The reference includes its tribute under a null/unknown group; the candidate drops it.

Nothing about the SQL changed. Only the data did. The useful question becomes:

**Can I construct a valid database state where these queries disagree?**

This is closely related to the approach in [SpotIt](https://proceedings.iclr.cc/paper_files/paper/2026/hash/70e692da44c19710386648694e2b899b-Abstract-Conference.html), an ICLR 2026 paper that uses bounded formal verification to search for databases that distinguish two SQL queries. We don't need a formal theorem prover in every eval harness. But the mental model is useful regardless.

## "Same result" hides several kinds of difference

The counterexample idea also explains why SQL comparison gets messy so quickly. Two queries can differ in ways that are cosmetic, contract-dependent, or genuinely semantic.

Some differences are mostly about representation:

* `tribute` vs. `annual_tribute` as an alias;
* columns returned in a different order;
* rows returned in a different order when ordering was never requested;
* `10` versus `10.0`;
* returning `province, governor` when the question only asked for the province.

A comparator can make explicit policy decisions about these.

Other differences change what happens for some legitimate data:

* `INNER JOIN` instead of `LEFT JOIN` drops provinces with no governor;
* filtering `IS NOT NULL` removes a valid unknown category;
* `DISTINCT` collapses separate tribute payments;
* a one-to-many join accidentally double-counts taxes;
* `> 1000` is used when the requirement says "at least 1000";
* `LIMIT 1` arbitrarily chooses one legion when several tie for largest;
* grouping by governor name merges two unrelated people both named Gaius;
* returning zero instead of "no matching row" changes the meaning of absence.

A single database snapshot may expose these differences. Or it may very politely conceal them.

I tightened my EX comparator’s handling of types, nulls, positions, duplicates and numeric equivalents. That made the deterministic metric better specified, but it changed zero historical EX outcomes. So comparator bugs were not the explanation for the big gap.

## EX fail, SQL still acceptable

The opposite disagreement is easier to imagine: sometimes EX is faithfully enforcing a contract that is stricter than the actual question.

Suppose the user asks:

> Which provinces supplied grain last year?

The reference returns only province names. The candidate returns province names plus the responsible governor. Under a strict positional comparator, EX fails. But the candidate did answer the question. It just brought paperwork. Or suppose the reference returns a count named `province_count` while the candidate calls it `num_provinces`.

Whether that should fail is not really a SQL question. It is a decision about the eval contract. At some point, correctness depends on the natural-language task:

**Did this SQL actually answer what the user asked?**

That is where an LLM judge becomes useful. It can consider the question, schema, SQL, and results together instead of comparing two arrays. But it is not a proof of SQL equivalence. It is another evaluator, with different strengths and failure modes.

## Don’t average away the disagreement

Once you have both kinds of graders, the obvious implementation is to use deterministic checks where possible and an LLM judge for cases that require interpretation.

Useful, yes. Interesting, not really.

The more useful lesson from my 38% versus 66% gap is to **preserve the disagreement categories instead of immediately collapsing them into one adjusted score**.

* Repeated **EX-pass / judge-reject** cases suggest missing counterexamples in your test data. Your fixture may simply be too friendly.
* Repeated **EX-fail / judge-accept** cases point somewhere else: comparator policy, gold answers, or an output contract that may be stricter than the user-facing task.

Those are different failure modes and they call for different fixes. So disagreement is not merely something to resolve before computing the final percentage. It is eval telemetry.

## This is not really about SQL

SQL makes the gap between output agreement and semantic correctness easy enough to see: we can execute both candidates and compare their outputs. But the same pattern shows up in agent evals more broadly.

A coding agent can pass every provided unit test while containing a bug in an untested branch. A browser agent can take exactly the expected sequence of actions and still fail the user's goal. Or it can take a completely different path and succeed.  A research agent can cite the expected sources while drawing the wrong conclusion—or use different sources and produce a well-supported answer.

In each case, deterministic graders answer questions like:

* Did this expected event occur?
* Did this output match?
* Did these tests pass?
* Did the agent use the required tool?

An LLM judge can ask a different question:

**Was the behavior actually acceptable for the task?**

That flexibility is valuable. It is also why the judge should not automatically become "ground truth". When the two disagree, inspect the disagreement before deciding which grader needs fixing.

## Back to EX vs. LLM judge (38% vs. 66%)

With that framing, the original discrepancy is less mysterious. I replayed the saved traces, fixed the score aggregation, tightened the EX comparator, reviewed the gold SQL and rejudged only the cases whose grading branch changed.

*The gap did not disappear.*

And that is the point. EX asks:

**Did these SQL queries produce the same result on the database I tested?**

The LLM judge asks something closer to:

**Does this SQL actually answer the question?**

Those are different questions.

For text-to-SQL—and for agent evals more broadly—the cases where they produce different answers are often more informative than either headline score on its own.
