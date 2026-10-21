---
layout: post
title: "Building a self-improvement loop"
---

# TLDR
{: .no_toc}

- I built a small self-improvement loop: one model reads my text-to-SQL agent's failures and the judge's reasoning, then edits the agent's prompt. Edits that raise the score are kept.
- 2 of 10 edits were kept. On fresh questions the agent went from 64% to 82%, and a second judge from another lab saw almost the same gain.
- One kept edit also moved the loop judge's reading of ambiguous questions into the prompt, and the second judge exposed it.

Experiment code: [exp_05_self_improvement_loop](https://github.com/nicpo/ai-agent-evals/tree/main/experiments/exp_05_self_improvement_loop)

<nav class="series-navigation" aria-label="AI agent evaluation series">
  <p><strong>Part of: <a href="/">Evaluating AI Agents</a></strong></p>
  <ol>
    <li><a href="{% post_url 2026-06-18-evals-for-ai-agents %}">Building the eval system</a></li>
    <li><a href="{% post_url 2026-07-10-graders-disagreement %}">Grader disagreement</a></li>
    <li><a href="{% post_url 2026-08-05-graders-agreement %}">When graders agree</a></li>
    <li><a href="{% post_url 2026-08-28-repeatability %}">Repeatability</a></li>
    <li><a href="{% post_url 2026-09-17-costs %}">Cost</a></li>
    <li><a href="{% post_url 2026-09-30-jev %}">Typed judge</a></li>
    <li aria-current="page">Self-improvement loop</li>
  </ol>
</nav>

# Contents
{: .no_toc}

* TOC
{:toc}

# Self-improvement loop

Self-improvement loops are everywhere now. They update what the agent reads: the prompt, the tool descriptions, its skills. One model reads the agent's failures and proposes a fix, and the fix stays if the score goes up. Whether that makes agents better, or just better at the eval, is still an open question.

I wanted to try it on [my agent]({% post_url 2026-06-18-evals-for-ai-agents %}), which answers questions about a small Roman Empire database by writing SQL. The post involves 4 models:

- **The agent** is a fast and cheap model (Anthropic's Claude Haiku 5.5). It writes SQL using 2 tools, `run_query` and `get_sample_rows`.
- **The loop judge** is a frontier reasoning model from another lab (OpenAI's GPT-6.1 Sol). It decides whether each answer is right, with a reference query as a guide. Its verdicts decide which edits the loop keeps.
- **The proposer** is a more capable model than the agent (Anthropic's Claude Sonnet 5.5). It reads the agent's failures and proposes edits.
- **The second judge** is an open-weights reasoning model from a third lab (Moonshot AI's Kimi K2 Thinking). It regrades answers after the loop, as a check on the loop judge.

The catch is that a loop doesn't optimize your agent. It optimizes a *score*. As a recent production report on these loops [puts it](https://arxiv.org/abs/2609.02246), "an optimizer converges to the decision boundary of whatever grades it, not to the boundary of the task". So the gain needs checking somewhere the loop can't reach.

# First, check what your harness grades

Before starting the loop, I read the baseline failures. Many weren't SQL mistakes. My harness grades the agent's *last* query, and the agent likes to double-check:

```sql
-- call 3: the right answer
SELECT p.name, SUM(t.amount_owed - t.amount_collected) AS shortfall ...
-- call 4: a sanity check, which became the graded "answer"
SELECT COUNT(*) FROM tribute_records WHERE amount_collected IS NULL;
```

One prompt line ("the last query you run must be the one that answers the question") took the agent from 67% to 75% on my 61 existing questions. I added it by hand and started the loop from there. Otherwise the loop's first win would have come from adapting to my harness, not from writing better SQL.

That line is a workaround. It doesn't stop the agent from checking its work. It just forces checks to happen before the answer query, and the agent kept slipping back (more below). The real fix belongs in the harness: a `submit_answer` tool or a marked final answer, so the agent can verify as much as it likes. If your harness grades "the last message" or "the final file state", check whether it's picking up the deliverable you intended.

# The loop

<figure class="project-map">
<figcaption>Flowchart: one round of the loop, and the checks used once after it</figcaption>
<div class="project-map-diagram">
  <div class="map-node map-node-primary">Current best prompt <span>plus 4 failed dev answers and the loop judge's reasoning</span></div>
  <div class="map-arrow" aria-hidden="true">↓</div>
  <div class="map-node">Proposer <span>diagnoses the failures, proposes 1 edit</span></div>
  <div class="map-arrow" aria-hidden="true">↓</div>
  <div class="map-node">Lint <span>rejects copied answers, values, reference SQL</span></div>
  <div class="map-arrow" aria-hidden="true">↓</div>
  <div class="map-node">Run on dev (30) and validation (31) <span>3 runs each, graded by the loop judge</span></div>
  <div class="map-arrow" aria-hidden="true">↓</div>
  <div class="map-node map-node-question">Validation +5 points, dev no worse than -5?</div>
  <div class="map-arrow" aria-hidden="true">↓</div>
  <div class="map-branch-row">
    <div class="map-branch">
      <div class="map-node map-node-agree">Keep</div>
      <p>Becomes the new best prompt.</p>
    </div>
    <div class="map-branch">
      <div class="map-node map-node-disagree">Reject</div>
      <p>Next round starts from the same prompt.</p>
    </div>
  </div>
  <p class="map-note">After the loop, used once: 24 fresh questions x 5 runs, regraded by a second judge.</p>
</div>
</figure>

My 61 existing questions are split in 2. The proposer sees failures from 30 dev questions, along with the loop judge's written reasoning. The other 31 validation questions decide which edits stay, and the proposer never sees them. A verdict like "the join drops officials with no records" tells it far more than "fail". That's the core idea of [GEPA](https://arxiv.org/abs/2507.19457), and the reason it can beat RL with far fewer runs.

That much detail has a downside: the easiest fix for a failed question is to write its answer into the prompt. So 2 checks run before an edit is evaluated. A lint rejects edits that smuggle in answers (a question id, a province name, a chunk of reference SQL). A 3,500-character cap stops the prompt from turning into a pile of patches, one per failure. 10 proposals cost $11.60.

# What the loop changed

The loop kept 2 edits. Between them they added 3 rules, all about how the agent works:

- **The final query must produce the answer set.** "Put every filter, eligibility condition, comparison ... ranking and LIMIT in the SQL". Before this, the agent often ran a broad per-group table and did the ranking in prose. Part of that is my harness again, since the judges only see the query. But ranking a dozen rows in its head is also where a model could make a mistake, plus it makes the agent's output harder to verify. So I'd want this rule with any judge.
- **"For each X", keep every X.** "For example with a LEFT JOIN from the table of X". More on this one below.
- **A pre-final check.** Before the last query, state the entity per row, the grain, the grouping columns ("only those the question names") and whether one row or many is expected.

None of them is something I'd have written on day one. All are obvious in hindsight, which is roughly what you want from a loop.

# The loop stopped at 2

The other 8 proposals all failed the keep rule (2 also lost too much on dev). The current best prompt didn't change, so every round got the *same 4* failed questions. The proposer itself labeled 2 of them judge strictness, e.g. a rounding objection that "the reference SQL has the same weakness". A third was the agent running a sanity check after its answer query, the habit my hand fix was meant to stop. 6 proposals tried to address that from other angles: a hard rule, an audit step before finishing etc. None produced a gain bigger than the noise: rerunning the unchanged starting prompt moved the validation score by 6.5 points on its own, more than my 5-point keep bar. By round 9 the proposer noted that "proposals 3-8 already tried to enforce 'answer query last', and none helped".

My read: the loop found the useful changes early. After that, it kept rewriting the prompt to fix failures that no prompt could fix. I didn't test the alternatives, but the next steps seem clear: resample failures each round, let the proposer drop ones it flags as judge strictness and move a failure that survives several prompt edits into the harness or a tool.

# 3 questions about the gain

Back to the catch from the start: the loop optimized a score, so the gain needs checking somewhere the loop couldn't reach.

## Does it hold on fresh questions?

Before the loop, I prepared 24 new questions (8 families of related variants), locked them and used them only at the end, running each 5 times.

| Prompt | Loop judge | Second judge (below) |
|---|---:|---:|
| Original | 56.7% | 55.8% |
| Hand-fixed (loop start) | 64.2% | 59.2% |
| Loop winner | 81.7% | 75.8% |
| Gain over hand-fixed, points (95% CI) | +17.5 (+5.8 to +30.0) | +16.7 (+8.3 to +26.7) |

Under the loop judge, the gain on fresh questions matches the gain on the 61 questions the loop trained on (+18.0). So it isn't memorization. It's also the first change in this series that's clearly bigger than the noise.

## Does another calibrated judge see it?

About 95% of passes depend on the loop judge's call (strict row matching rejects any extra column). So the gain could reflect the loop judge's opinion as much as the agent's skill, and I needed a second judge outside the loop. I calibrated 5 open-weights models from labs other than the loop judge's against the 196 cases I'd labeled for the [Jev post]({% post_url 2026-09-30-jev %}). The best became the second judge, Kimi K2 Thinking: it agreed with me 94% of the time. The other 4 were Kimi K2.5, GLM 4.7, DeepSeek V3.2 and Qwen3 Next 80B. Regrading ~900 saved answers cost $5.35.

On fresh questions the judges agree. On the 61 questions the loop trained on, they don't: the loop judge gives the winner 92%, the second judge 80%. I read the 25 answers the loop judge passed and the second judge failed. In 8, the loop judge let real errors through. In 17, I sided with the loop judge, and 9 of those came from one habit: the LEFT JOIN rule. A fictional version (the real questions stay private):

```sql
-- "For each legion, how many soldiers deserted?"
-- Reference: only legions with desertion records
SELECT l.name, COUNT(*) FROM legions l JOIN desertions d ON ...
-- Agent after the loop: every legion, 0 where there are none
SELECT l.name, COUNT(d.id) FROM legions l LEFT JOIN desertions d ON ...
```

The loop judge had once failed an answer for dropping entities. The proposer considered whether that was ambiguity, decided that "'for each official' reasonably implies all officials" and wrote the rule. The loop judge accepts both readings, while the second judge rejects the LEFT JOIN. I personally think the LEFT JOIN is defensible here.

So the gap splits roughly in thirds:

- the loop judge letting errors through
- the second judge being strict about things like extra columns
- the LEFT JOIN habit

The last third is the quiet one: the loop absorbed its judge's reading of questions with more than one fair answer. Automated checks like the lint wouldn't catch this, because nothing broke. A second judge shows you where it happened. Who's right is up to whoever owns the question: reword it, or tell the judges both readings pass.

## What changed in behavior and cost?

On the 24 fresh questions the loop never saw, the winner explores more: 5.1 tool calls per attempt instead of 4.2, with more sample-row lookups, and failed queries fell from 0.14 to 0.11 per attempt. That's a change in how it writes SQL, not just the last-query rule.

| Agent cost | Hand-fixed | Loop winner |
|---|---:|---:|
| Per 1,000 attempts | $1.60 | $1.90 |
| Per 1,000 correct answers (loop judge) | $2.50 | $2.30 |

It costs ~20% more per attempt and slightly less per correct answer. I didn't record latency.

# Building a loop for your agent

So was it worth it? For about $25 all-in ($12 of it the loop itself), the agent got ~17 points better on questions it never saw, at about the same cost per correct answer. It worked because I already had an eval I could rerun automatically, with a judge that explains its verdicts. The failures also kept recurring, and they came from the agent's habits (where it puts the logic, what it checks before answering), which a prompt can change. If your setup looks like that, this is the sequence I'd follow:

1. **Start with failures you understand.** Fix harness mismatches by hand first.
2. **Let a proposer edit a narrow surface**, like the prompt and tool descriptions, with a length cap and a budget.
3. **Select edits on tasks the proposer doesn't see.**
4. **If it stalls, check which failures it keeps trying to fix.** Some failures need a harness or tool change, not another instruction.
5. **Check the winner on fresh tasks.** Where correctness takes judgment, regrade with a second calibrated judge.
6. **Read the disputed cases and compare cost per correct answer** before you ship the change.

If you don't trust your judge yet, the loop will learn its mistakes along with everything else. Fix the judge before building the loop, by checking its verdicts against answers you've labeled yourself ([how I checked mine]({% post_url 2026-07-10-graders-disagreement %})).

None of this is SQL-specific. A support bot tuned against a judge that likes apologies will learn to apologize more. Whether that's improvement depends on whether you share the judge's taste, and you only find out by looking.

<nav class="series-pagination" aria-label="AI agent evaluation series navigation">
  <a href="{% post_url 2026-09-30-jev %}">← Previous: Typed judge</a>
  <a href="/">All experiments →</a>
</nav>
