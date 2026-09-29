---
layout: post
title: "Measuring agent reliability with repeated evals"
---

# TLDR
{: .no_toc}

- My earlier evals of a text-to-SQL agent ran each question once
- I re-ran three versions of the agent on 61 questions, three times each: 549 attempts at temperature 0
- The agent sometimes wrote different SQL for the same question, and the LLM judge sometimes graded the same SQL differently
- Most questions weren't coin flips passed 3/3 or failed 0/3, so retries recovered almost nothing.
- But the average improvement hid a reshuffle: prompt v3 gained on medium and hard questions and lost on easy ones.

Experiment code: [exp_03_repeatability_costs](https://github.com/nicpo/ai-agent-evals/tree/main/experiments/exp_03_repeatability_costs)

<nav class="series-navigation" aria-label="AI agent evaluation series">
  <p><strong>Part of: <a href="/">Evaluating AI Agents</a></strong></p>
  <ol>
    <li><a href="{% post_url 2026-06-18-evals-for-ai-agents %}">Building the eval system</a></li>
    <li><a href="{% post_url 2026-07-10-graders-disagreement %}">Grader disagreement</a></li>
    <li><a href="{% post_url 2026-08-05-graders-agreement %}">When graders agree</a></li>
    <li aria-current="page">Repeatability</li>
    <li><a href="{% post_url 2026-09-17-costs %}">Cost</a></li>
  </ol>
</nav>

* TOC
{:toc}

# Recap on the agent, and why I ran it again

I've been building an eval harness for a text-to-SQL agent. It answers questions about a small database of the Roman Empire: provinces, officials, decrees and tribute records. The eval set has 61 questions, graded by execution accuracy (does the result match the gold query's result?) and by an LLM judge.

In the [first post](https://nicpo.github.io/2026/06/18/evals-for-ai-agents), I compared three versions of the agent:

- **v1** works out the database structure by looking at sample rows.
- **v2** gets a tool that returns the schema.
- **v3** has the schema pasted into its system prompt, and loses the schema tool.

```text
v1: question -> peek at sample rows -> write SQL -> run SQL -> maybe revise
v2: question -> fetch schema        -> write SQL -> run SQL -> maybe revise
v3: question + schema               -> write SQL -> run SQL -> maybe revise
```

The idea behind v3 was that if the agent already has the schema, it shouldn't need to rediscover it every time it's called. And it didn't. Across the new runs, v3 did less work:

| Version | Tool calls per attempt | LLM calls per attempt | Tokens per attempt |
| --- | ---: | ---: | ---: |
| v1 | 5.1 | 4.9 | 9.3k |
| v2 | 2.8 | 3.6 | 10.3k |
| v3 | 1.7 | 2.5 | 8.7k |

That's 66% fewer tool calls and 49% fewer LLM calls than v1. Tokens and cost are less clear: v3 is clearly cheaper than v2, but the difference from v1 could still be noise.

The open question was quality. After I fixed the scoring formula in my [second post](https://nicpo.github.io/2026/07/20/ex-vs-llm-judge), the single run said v3 paid for its efficiency: it got 67% of questions right, against 80% for v1. But that was one run per question, so I ran the whole thing again, three times over:

```text
61 questions x 3 versions x 3 runs = 549 attempts
```

The setup was the same as before:

* Agent: Claude Haiku 4.5 as the agent at temperature 0
* Judge: GPT-5.4 as the judge
* 12-tool call limit per attempt
* Same database


# Accuracy flipped

| Version | Accuracy - single run | Accuracy - 3-run mean |
| --- | ---: | ---: |
| v1 | 80% | 71% |
| v2 | 67% | 73% |
| v3 | 67% | 78% |

v3 went from 13 points behind v1 to 7 points ahead. That flipped the ranking. Why the change?

Between the two posts I corrected two gold queries:

- One edit was cosmetic. It removed a column the question never asked for.
- The other made the gold includes rows the old one dropped (a left join instead of an inner join).

The second edit cost v2 and v3 a couple of successes each on that question and left v1 about where it was. So the gold changes pushed *against* v3's rise, not for it.

That leaves randomness, but randomness doesn't explain all of it. I can build 27 "single-run" results from the new data by picking one of the three runs for each version. In every one of them, v3 beats v1, by 3 to 11 points. None comes close to v1 leading by 13. So the earlier single run wasn't just an unlucky draw from the same distribution. Something else also shifted, and I haven't been able to pin it down. A hosted model can change behind an unchanged name, and so can the infrastructure serving it.

This matches [*On Randomness in Agentic Evals*](https://arxiv.org/abs/2602.07150) (Bjarnason, Silva and Monperrus). Across 60,000 agent trajectories, picking a different single run moved pass@1 by 2.2 to 6.0 points, even at temperature 0. The authors' conclusion: an improvement of 2 to 3 points from a single run may be pure noise.

# Metrics for evaluating several runs

With several runs per question, we can measure "how good is it?" with three metrics:

- **Mean success (pass@1):** the share of all attempts that succeeded. The expected result of a single try.
- **pass^k:** the share of questions the agent got right on *every* one of k tries. How often you can count on it.
- **pass@k:** the share of questions it got right *at least once* in k tries. How much retries could cover.

Here `k = 3`, so from now on I'll write pass^3 and pass@3.

| Version | Mean success | pass^3 | pass@3 |
| --- | ---: | ---: | ---: |
| v1 | 71.0% | 63.9% | 77.0% |
| v2 | 73.2% | 68.9% | 78.7% |
| v3 | 77.6% | 72.1% | 83.6% |

Which number matters depends on how the agent is deployed:

- **The user gets one answer.** Mean success should be the headline. pass^3 tells you how many questions you can rely on.
- **You can retry and can tell which answer is right** (a test suite, a validator, a human check). Then pass@3 is a real success rate.
- **You can retry but can't tell which answer is right.** Then pass@3 is a ceiling you'll never reach.

The three numbers sit close together because most questions don't wobble. For v3, 44 questions passed 3/3, 10 failed 0/3, and only 7 landed in between. Most questions aren't coin flips, that's why retries barely helped.

<img src="/assets/2026-08-28-repeatability/per-question-reliability.png" alt="Per-question reliability by agent version" style="max-width: 800px; width: 100%; height: auto;">

The overall pattern is visible in this picture:

<img src="/assets/2026-08-28-repeatability/correctness-by-difficulty-heatmap.png" alt="Adjusted correctness by difficulty and agent version" style="max-width: 700px; width: 100%; height: auto;">

# Variance in agent and grader output across runs

A score comes out of a pipeline with two moving parts: the agent writes SQL, then the judge grades it. Either can vary.

<img src="/assets/2026-08-28-repeatability/sources-of-variability.png" alt="Adjusted correctness by difficulty and agent version" style="max-width: 1000px; width: 100%; height: auto;">

## The agent

Temperature 0 doesn't mean deterministic. Even with identical inputs, the agent wrote the same SQL on all three runs (ignoring whitespace, casing and comments) for only:

- 40 of 61 questions in v1
- 49 of 61 in v2
- 49 of 61 in v3

v1 wobbles the most, and it also takes the longest path. My guess is that every extra step is another fork where two runs can go different ways. [Thinking Machines Lab](https://thinkingmachines.ai/blog/defeating-nondeterminism-in-llm-inference/) explains where the randomness comes from in the first place. Inference servers batch requests together, and a model's arithmetic gives slightly different results depending on how full the batch is. Your request's output depends on who else is using the API at that moment.

Different SQL isn't necessarily worse SQL, since there are many correct ways to write a query. And identical SQL isn't necessarily right. On one easy question, v2 failed 3 out of 3 while v1 and v3 passed 3 out of 3. Here's an example of the same kind of mistake:

```sql
-- "How many decrees were issued each year?"
SELECT year, COUNT(*) FROM decrees
WHERE province_id IS NOT NULL   -- added "to be safe"
GROUP BY year;
```

The defensive filter in fact drops every empire-wide decree, since those apply to no single province. A single run would have logged this as one miss. Three identical misses reveal a habit. Perhaps reading the schema's nullable columns nudged v2 toward filtering them.

## The judge

The judge is an LLM too. So freezing its model and prompt didn't freeze its opinions. The same question and the same normalized SQL were graded more than once in 146 cases. In 8 of those, the verdicts disagreed.

In the starkest case, one hard question got identical SQL nine times. The judge passed it six times and *failed it three times*, once in each version. On another question, identical SQL got `ACCEPTABLE` in all three v3 runs, but only once in three v2 runs.

So a repeated eval asks two separate questions:

1. Did the **agent** produce something different?
2. Did the **judge** grade the same thing differently?

They need different fixes. Agent variation is a property of your product: measure it, report it and decide whether to reduce it. Judge variation should be fixed in the harness. For example, cache verdicts by (question, normalized SQL) so identical inputs always get the same grade, or run the judge several times and take the majority.

# The average hid a reshuffle

v3 improved accuracy by +7 points. Underneath, the outcomes churned. Lining up v1 and v3 attempt by attempt (same question, same run number):

|  | v3 right | v3 wrong |
| --- | ---: | ---: |
| **v1 right** | 106 | 24 |
| **v1 wrong** | 36 | 17 |

v3 won 36 attempts that v1 lost, and lost 24 that v1 won. That's 60 flips for a net gain of 12.

Moreover, those flips weren't random. They sorted by difficulty:

| Difficulty | Questions | v1 | v3 | v3 minus v1 (95% interval) |
| --- | ---: | ---: | ---: | ---: |
| Easy | 15 | 93% | 80% | -13 (-40 to +13) |
| Medium | 20 | 70% | 88% | +18 (0 to +38) |
| Hard | 16 | 54% | 75% | +21 (-6 to +48) |
| Extra-hard | 10 | 67% | 57% | -10 (-43 to +17) |

(The intervals come from resampling whole questions, so three runs of one question count as one data point, not three.)

Most intervals include zero, so this is a pattern that needs to to be further tested. But it's a pattern with a mechanism:

- **Medium and hard questions** are where v1 explored most, averaging 7.2 tool calls per hard question against v3's 2.3. The schema costs a fixed number of tokens on every call, and it pays off when the alternative is a long walk through the database.
- **Easy questions** are where v1 barely needed to explore, so v3's schema was pure overhead: more tokens, not better answers. And the v3 easy-question failures I dug into had nothing to do with the schema. They were misreadings of the question. Here's the kind of mistake it made:

```sql
-- "Which provinces are imperial provinces?"
SELECT name FROM provinces WHERE status LIKE '%imperial%';
-- also matches 'formerly imperial'; the question wanted status = 'imperial'
```

Knowing the schema didn't help it read the question correctly.

Here's the quality difference between v1 and v3 by difficulty. The dot = v3 - v1. Line = 95% range (if the line crosses zero, the difference could be noise).

<img src="/assets/2026-08-28-repeatability/v3-v1-quality-differences-by-difficulty.png" alt="v3-v1 quality differences by difficulty" style="max-width: 1000px; width: 100%; height: auto;">

## Tool call limits

On hard questions, v1 sometimes explored until it hit the wall. 14 of its 183 attempts (8%) reached the 12-tool-call limit without finishing. v2 and v3 never hit it. One hard question failed 0/3 for v1 and passed 3/3 for both other versions. On another, v1's last query was actually correct, but the agent never stopped to say so.

That raises two different questions:

1. **Did the agent, as deployed, succeed within its budget?** No -> it's a failure.
2. **Could the model have got there with more room?** I didn't explore this here.

The headline metric should count budget exhaustion as a failure, because the prompt, tools, loop and limits together *are* the agent. But report it separately too. "Wrote the wrong SQL" and "ran out of room while exploring" need different fixes.

# So, did the schema in the prompt work?

- Work: yes, clearly. v3 made 66% fewer tool calls, 49% fewer LLM calls and no budget failures compared to v1.
- Quality: probably no worse. v3 led v1 in each of the three runs, and on average by 7 points. But the interval runs from a 7-point loss to a 20-point gain, so three runs can't rule out a modest regression.

The broader lesson is about **context strategy**. Loading high-value context upfront trades a fixed token cost for fewer exploratory steps. It works here because the database has four small tables. A warehouse with 400 tables would bury every question under tens of thousands of tokens of schema, and a fast-changing schema would go stale in the prompt. There, what works better is loading some context upfront for speed, and letting the agent [retrieve the rest when it needs it](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents).

# Takeaways

**Run it more than once, and in one go.** A single run can give a ranking that won't survive a rerun. And a rerun months later might well be a different experiment, even when model versions are fixed and temperature is 0.

**Count questions, not attempts.** Three runs of 61 questions is still 61 questions. Compute uncertainty by resampling whole questions, as Evan Miller recommends in [*Adding Error Bars to Evals*](https://arxiv.org/abs/2411.00640).

**Match the metric to the deployment:**
* One shot in production: `mean success`
* Retries with a verifier: `pass@k`
* Anything that must work every time: `pass^k`

[τ-bench](https://arxiv.org/abs/2406.12045) introduced pass^k for exactly this reason. Agents that looked fine on pass@1 fell apart when asked to succeed repeatedly.

**Separate the agent's variability from the judge's.** If an LLM grades your outputs, make sure identical outputs get identical grades. Otherwise part of what you report as agent variance is measurement error.

**Break the average down before calling a change neutral.** Here, +7 points came from 36 wins and 24 losses in different parts of the eval set. A change that helps hard cases and hurts easy ones can look like a modest win overall.

**None of this is specific to SQL.** A coding agent, a support bot or a research agent faces the same questions. Does it give the same answer twice? Does your grader? Does "better on average" mean better everywhere? And the context trade-off carries over too: a schema, an API spec or a style guide in the prompt saves exploration when it's small and stable, and gets in the way when it isn't.

<nav class="series-pagination" aria-label="AI agent evaluation series navigation">
  <a href="{% post_url 2026-08-05-graders-agreement %}">← Previous: When graders agree</a>
  <a href="{% post_url 2026-09-17-costs %}">Next: Cost →</a>
  <a href="/">All experiments →</a>
</nav>
