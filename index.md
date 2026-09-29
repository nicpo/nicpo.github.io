---
layout: default
title: "Evaluating AI Agents"
---

# Evaluating AI Agents: a text-to-SQL case study
{: .no_toc}

*I built an end-to-end evaluation system for a text-to-SQL agent, then used it as a testbed to investigate a harder question: how much can we trust an agent eval?*

## The system

<img src="/assets/root/system-diagram.png" alt="System diagram" style="max-width: 300px; width: 100%; height: auto;">

The harness runs a hand-reviewed set of questions, saves each agent trace, grades what happened at several levels, and feeds the results back into the next agent version.

## What I investigated

1. Building an eval harness and calibrated graders
2. What deterministic-vs-semantic grader disagreement tells us
3. How both graders can miss the same failure
4. How repeated runs change conclusions about agent quality
5. How architecture changes affect inference cost

<figure class="project-map">
<figcaption>Can we trust the eval?</figcaption>
<div class="project-map-diagram">
  <div class="map-node">Build eval</div>
  <div class="map-arrow" aria-hidden="true">↓</div>
  <div class="map-node map-node-primary">Agent</div>
  <div class="map-arrow" aria-hidden="true">↓</div>
  <div class="map-grader-row">
    <div class="map-node">Deterministic graders</div>
    <div class="map-node">LLM judge</div>
  </div>
  <div class="map-arrow" aria-hidden="true">↓</div>
  <div class="map-node map-node-question">Do the verdicts agree?</div>
  <div class="map-branch-row">
    <div class="map-branch">
      <div class="map-node map-node-disagree">Disagree</div>
      <p>Why? What does each measure?</p>
    </div>
    <div class="map-branch">
      <div class="map-node map-node-agree">Agree</div>
      <p>Are they both still wrong?</p>
    </div>
  </div>
  <div class="map-arrow" aria-hidden="true">↓</div>
  <div class="map-node map-node-primary">Repeatability <span>Do results persist?</span></div>
  <div class="map-arrow" aria-hidden="true">↓</div>
  <div class="map-node">Cost <span>Is the improvement worth it?</span></div>
</div>
</figure>

## Selected findings

- 61 hand-reviewed evaluation questions, spanning four difficulty levels.
- 5 grader layers, combining deterministic checks with LLM judgment.
- 352 counterfactual query-and-database comparisons exposed failures that an original-data evaluation could not see.
- 549 repeated agent runs changed the ranking of three agent versions, despite temperature 0.
- **49% fewer LLM calls** translated to only an estimated **11% lower inference cost**. That cost reduction is uncertain: the 95% interval ranges from a **$3.04 saving to a $0.53 increase per 1,000 attempts**.

## Experiments

<div class="experiment-cards">
  <a class="experiment-card" href="{% post_url 2026-06-18-evals-for-ai-agents %}">
    <h3>1. Build the eval</h3>
    <p>Build the harness, a 61-question eval set, trace capture, and a layered grading system for a text-to-SQL agent.</p>
    <p>Read: Building and scaling evals for AI agents →</p>
  </a>
  <a class="experiment-card" href="{% post_url 2026-07-10-graders-disagreement %}">
    <h3>2. Investigate disagreement</h3>
    <p>Use the gap between execution accuracy and an LLM judge to find out what each measure actually observes.</p>
    <p>Read: Lessons from the mismatch between your deterministic grader and the LLM judge →</p>
  </a>
  <a class="experiment-card" href="{% post_url 2026-08-05-graders-agreement %}">
    <h3>3. Test agreement</h3>
    <p>Probe accepted SQL against valid changes to the database—and find a failure both graders missed.</p>
    <p>Read: When the graders agree, they may not have seen everything →</p>
  </a>
  <a class="experiment-card" href="{% post_url 2026-08-28-repeatability %}">
    <h3>4. Measure repeatability</h3>
    <p>Repeat the full eval to separate a durable quality difference from one-run noise and hidden model variation.</p>
    <p>Read: Measuring agent reliability with repeated evals →</p>
  </a>
  <a class="experiment-card" href="{% post_url 2026-09-17-costs %}">
    <h3>5. Account for cost</h3>
    <p>Measure calls, tokens, and price together to test whether a simpler architecture is actually cheaper.</p>
    <p>Read: Fewer agent steps do not mean proportionally lower cost →</p>
  </a>
</div>

## What this project exercises

`Agent evaluation · experimental design · LLM-as-judge · SQL · Python · statistical analysis · failure analysis · inference economics`

## Implementation

The experiment code is in [github.com/nicpo/ai-agent-evals](https://github.com/nicpo/ai-agent-evals).
