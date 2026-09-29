---
layout: default
title: "Evaluating AI Agents"
---

# Evaluating AI Agents: a text-to-SQL case study
{: .no_toc}

*I built an end-to-end evaluation system for a text-to-SQL agent, then used it as a testbed to investigate a harder question: how much can we trust an agent eval?*

## The system

<div class="diagram-placeholder" role="note">
  <strong>System diagram placeholder</strong><br>
  Project image to be added here: 61-question eval set → text-to-SQL agent → trace → deterministic graders + LLM judge → evaluation → agent iteration.
</div>

The harness runs a hand-reviewed set of questions, saves each agent trace, grades what happened at several levels, and feeds the results back into the next agent version.

## What I investigated

1. Building an eval harness and calibrated graders
2. What deterministic-vs-semantic grader disagreement tells us
3. How both graders can miss the same failure
4. How repeated runs change conclusions about agent quality
5. How architecture changes affect inference cost

<figure class="project-map">
<figcaption>Can we trust the eval?</figcaption>
<pre>                         Build eval
                             │
                             ▼
             ┌── deterministic graders
Agent ───────┤
             └── LLM judge
                             │
                ┌────────────┴────────────┐
                ▼                         ▼
           DISAGREE                    AGREE
                │                         │
          Why? What does            Are they both
          each measure?             still wrong?
                │                         │
                └────────────┬────────────┘
                             ▼
                       REPEATABILITY
                   Do results persist?
                             │
                             ▼
                           COST
                 Is the improvement worth it?</pre>
</figure>

## Selected findings

- 61 hand-reviewed evaluation questions, spanning four difficulty levels.
- 5 grader layers, combining deterministic checks with LLM judgment.
- 352 counterfactual query-and-database comparisons exposed failures that an original-data evaluation could not see.
- 549 repeated agent runs changed the ranking of three agent versions, despite temperature 0.
- **49% fewer LLM calls** translated to only an estimated **11% lower inference cost**. That cost reduction is uncertain: the 95% interval ranges from a **$3.04 saving to a $0.53 increase per 1,000 attempts**.

## Experiments

<div class="experiment-cards">
  <section class="experiment-card">
    <h3>1. Build the eval</h3>
    <p>Build the harness, a 61-question eval set, trace capture, and a layered grading system for a text-to-SQL agent.</p>
    <p><a href="{% post_url 2026-06-18-evals-for-ai-agents %}">Building and scaling evals for AI agents</a></p>
  </section>
  <section class="experiment-card">
    <h3>2. Investigate disagreement</h3>
    <p>Use the gap between execution accuracy and an LLM judge to find out what each measure actually observes.</p>
    <p><a href="{% post_url 2026-07-10-graders-disagreement %}">Lessons from the mismatch between your deterministic grader and the LLM judge</a></p>
  </section>
  <section class="experiment-card">
    <h3>3. Test agreement</h3>
    <p>Probe accepted SQL against valid changes to the database—and find a failure both graders missed.</p>
    <p><a href="{% post_url 2026-08-05-graders-agreement %}">When the graders agree, they may not have seen everything</a></p>
  </section>
  <section class="experiment-card">
    <h3>4. Measure repeatability</h3>
    <p>Repeat the full eval to separate a durable quality difference from one-run noise and hidden model variation.</p>
    <p><a href="{% post_url 2026-08-28-repeatability %}">Measuring agent reliability with repeated evals</a></p>
  </section>
  <section class="experiment-card">
    <h3>5. Account for cost</h3>
    <p>Measure calls, tokens, and price together to test whether a simpler architecture is actually cheaper.</p>
    <p><a href="{% post_url 2026-09-17-costs %}">Fewer agent steps do not mean proportionally lower cost</a></p>
  </section>
</div>

## What this project exercises

`Agent evaluation · experimental design · LLM-as-judge · SQL · Python · statistical analysis · failure analysis · inference economics`

## Implementation

The experiment code is in [github.com/nicpo/ai-agent-evals](https://github.com/nicpo/ai-agent-evals).
