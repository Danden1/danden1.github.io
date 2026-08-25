---
title: "Trying GEPA on My AI Agent"
description: "I switched my agent to a smaller model, and the old prompt stopped working reliably. This post explains how GEPA works and why applying it was harder than I expected."
date: 2026-08-25 00:10:00 +0900
categories: [AI, Agent]
tags: [GEPA, Prompt Optimization, AI Agent, Paper]
---

## Background

I am building an AI agent. To reduce inference cost, I replaced its language model with a smaller one. The agent still ran, but the old prompt became less reliable. The new model sometimes missed conditions that were already written in it.

I could keep editing the prompt and rerunning the same scenarios, but I wanted a more systematic approach.

That was why I started looking into automated prompt optimization and found [GEPA: Reflective Prompt Evolution Can Outperform Reinforcement Learning](https://arxiv.org/abs/2507.19457).

GEPA does not update the task model's weights. It collects execution traces and evaluation feedback, then asks a reflection model to propose a prompt change. It keeps the change only when the score improves.

This avoids weight training, but it is not free. GEPA still requires task-model rollouts, evaluator calls, validation runs, and calls to the reflection model.

Across six tasks, the paper reports that GEPA beat GRPO by 6% on average and by up to 20%. It also used up to 35 times fewer rollouts. These results come from the paper's experimental settings. They do not guarantee that every GEPA run will be cheaper or faster.

I have not completed the experiment on my agent yet. This post covers the algorithm, what I initially misunderstood, and what I still need to apply it.

## How GEPA works

The paper first summarizes the complete loop in Figure 3.

![Overview of the GEPA optimization loop](/assets/img/posts/gepa/gepa-overview.png)
_The complete GEPA workflow from the [GEPA paper](https://arxiv.org/abs/2507.19457), Figure 3._

The left side shows the candidate pool and per-example scores. The middle applies the Pareto filter. The right side proposes a candidate, tests it on a minibatch, and either discards it or evaluates it on the full selection set.

Figure 3 also shows GEPA+Merge, an optional path that combines two candidate lineages. This post follows the standard reflective-mutation path in Algorithm 1.

Figure 4 provides the pseudocode behind that overview.

![GEPA's core optimization loop and Pareto-based candidate selection](/assets/img/posts/gepa/gepa-algorithms-1-2.png)
_Algorithm 1 and Algorithm 2 from the [GEPA paper](https://arxiv.org/abs/2507.19457), Figure 4._

- **Algorithm 1** is the main optimization loop. It evaluates the seed prompt, selects a parent, generates a new prompt, and decides whether to keep it.
- **Algorithm 2** only selects the parent candidate for the next iteration.

I first mixed up these two roles. Pareto selection does not decide whether a new prompt can enter the pool. It chooses which existing candidate to modify next. A separate minibatch comparison decides whether the new candidate is accepted.

For the core loop, think of a candidate as one prompt version and a task instance as one evaluation example. The paper also supports systems with several LLM prompts. That only matters at `SelectModule`, which I explain later.

### 1. Split the input data by purpose

Algorithm 1 receives one dataset named `D_train`. Its first line partitions that input into two subsets: `D_feedback` and `D_pareto`. This is a data split performed before the optimization loop. It is not an LLM classifying examples during optimization.

The pseudocode fixes the size of `D_pareto` with the `n_pareto` hyperparameter, but it does not specify whether the split should be random, stratified, or constructed by another sampling rule.

The two subsets have different jobs.

- `D_feedback` provides examples for prompt mutation. GEPA samples a small minibatch from it, runs the selected parent, and sends the resulting inputs, outputs, traces, scores, and feedback to the reflection model.
- `D_pareto` provides the per-example scores used to compare candidates, build the Pareto winner sets, and select the final prompt.

There are three similar but distinct descriptions in the paper and the current library:

| Context | Feedback data | Pareto selection data |
|---|---|---|
| Algorithm 1 pseudocode | A subset split from `D_train` | Another subset split from `D_train`, with size `n_pareto` |
| Paper experiments | The benchmark's training split | The benchmark's validation split |
| Current GEPA API | `trainset` | `valset`; if omitted, the implementation reuses `trainset` |

The paper's Appendix E.4 explicitly says that its experiments use the benchmark training set as `D_feedback` and the validation set as `D_pareto`. A separate test set is used only for final reporting.

The pseudocode and the reported experiments do not describe the split in exactly the same way. For my implementation, I still need data for two roles. The reflection model needs examples to learn from. GEPA also needs examples for candidate selection.

The official FAQ recommends an 80/20 train-validation split when there are more than 200 examples. It recommends a 50/50 split when there are fewer than 200. This is documentation guidance, not a rule in Algorithm 1.

### 2. Evaluate the seed prompt

GEPA starts with a seed candidate. In the simplest setup, this is the original system prompt.

```python
seed_candidate = {
    "system_prompt": "You are a recommendation agent...",
}
```

Before proposing any changes, GEPA runs the seed candidate on every example in `D_pareto`. The evaluator produces one numeric score per example. These scores become the first row of what is effectively a candidate-by-example score table.

The score does not have to come from an LLM. It comes from the evaluator supplied by the user. I will return to this because evaluator design is likely to be the most difficult part of my experiment.

### 3. Select a parent with the Pareto algorithm

Algorithm 2 selects one existing candidate to use as the parent of the next mutation. Every candidate considered here is already in the candidate pool. This algorithm does not control whether a new child is admitted to the pool.

The selector does not simply choose the candidate with the highest average score. It uses the per-example scores recorded on `D_pareto`.

Consider the following simplified score table. These values are only an example for explaining the algorithm; they are not results from the paper.

| Candidate | Case 1 | Case 2 | Case 3 | Average |
|---|---:|---:|---:|---:|
| `P0` | 1.0 | 0.0 | 0.5 | 0.50 |
| `P1` | 1.0 | 1.0 | 0.3 | 0.77 |
| `P2` | 0.8 | 0.7 | 0.9 | 0.80 |

The best candidates for each case are:

```text
Case 1: P0 and P1
Case 2: P1
Case 3: P2
```

Ties are kept. This means `P0` and `P1` are both winners for Case 1. Algorithm 2 takes the union of these per-example winner sets to form the initial parent-selection set.

The word `dominated` can be misleading here. A candidate is not removed just because its average score is lower. The [official implementation](https://github.com/gepa-ai/gepa/blob/b265bf9ca77fd8e8d82039d9f74911b8780fe1ce/src/gepa/gepa_utils.py#L40-L128) checks whether every case it wins still has another surviving winner. If so, the candidate can be removed. The process always leaves at least one winner for every case. It checks candidates with lower aggregate scores first.

In the example, `P0` wins only Case 1, and `P1` is another winner for that case. Removing `P0` does not leave Case 1 without a winning candidate, so `P0` can be removed from the parent-selection set. `P1` cannot then be removed because it is the only remaining winner for Case 2. `P2` is the only winner for Case 3.

After pruning, the surviving set contains `P1` and `P2`. GEPA counts how many remaining per-example winner sets contain each candidate. `P1` appears twice and `P2` appears once. The implementation builds a sampling list equivalent to `[P1, P1, P2]` and makes one random choice. `P1` therefore has a two-thirds chance of becoming the parent, while `P2` has a one-third chance.

This mechanism preserves different strategies. `P2` has the best overall average, but GEPA does not spend the entire budget modifying only `P2`. A candidate that is especially good at a smaller group of cases still gets a chance to evolve.

The probability refers only to parent selection. It is not the probability that a candidate will enter the pool. A new child enters the pool only after the later minibatch acceptance test. The full candidate history is also not deleted when a candidate is pruned here; pruning changes the parent-selection view.

### 4. Select the prompt to update

After Algorithm 2 has chosen a parent, line 8 of Algorithm 1 calls `SelectModule` on that parent. This is where round-robin appears.

Round-robin does not select the parent again. It selects which prompt to update inside the chosen parent.

This step exists because the paper also covers systems with several LLM calls. For example, a query rewriter and an answer generator may use different prompts. The paper calls each LLM component a module.

```text
Pareto candidate selection: Which candidate should be the parent?
Round-robin module selection: Which prompt in that parent should be updated?
```

If a system has three prompts, round-robin moves through them in order:

```text
query_rewriter
-> product_ranker
-> recommendation_explainer
-> query_rewriter
-> ...
```

The current implementation remembers the next prompt to update for each candidate. It advances that position after every selection. This stops the default policy from updating only the first prompt.

If the system has only one prompt, round-robin has no practical effect. It selects the same prompt every time. The current library can also update all prompts together, but Algorithm 1 updates one at a time.

### 5. Sample a feedback minibatch

GEPA samples `b` examples from `D_feedback`. The paper uses a minibatch size of three in its experiments, and the current implementation also uses three as the default for the standard optimization mode.

The selected parent runs on these examples. GEPA collects the information needed to explain what happened, such as:

- the input passed to the agent;
- the generated output;
- intermediate tool calls or module outputs;
- the numeric score;
- parsing errors or constraint violations;
- textual feedback from the evaluator.

The paper separates model-generated execution traces from evaluation traces produced by the environment. A reasoning step or tool call belongs to the execution trace. A compiler error, failed rubric item, or schema-validation error belongs to the evaluation trace. Both can give the reflection model more information than a score alone.

### 6. Evaluate the outputs with a custom evaluator

GEPA requires a numeric score where a higher value means a better result, but it does not require one specific scoring method. The evaluator belongs to the application, so it can be implemented in several ways.

#### Deterministic checks

The simplest option is hard-coded evaluation. The current `DefaultAdapter` includes a `ContainsAnswerEvaluator`, which checks whether the expected answer string appears in the model response.

The same pattern can be extended to checks such as:

- whether the response contains a required phrase;
- whether the output is valid JSON;
- whether every required field exists;
- whether the agent selected a valid tool;
- whether a product ID exists in the supplied catalog;
- whether the answer stays within a token or character limit.

These checks are cheap, repeatable, and easy to debug. They work well for hard constraints, but they cannot reliably judge whether a recommendation is thoughtful or whether an explanation is well grounded.

#### LLM-as-a-judge

For subjective requirements, the evaluator can call another LLM. The official FAQ recommends this evaluator-optimizer pattern for fuzzy or creative tasks: an LLM judge returns both a score and detailed feedback, and GEPA optimizes against them.

For example, a judge could assess whether a recommendation:

- respects the user's stated request;
- uses the available preference history appropriately;
- explains the trade-off between price and preference;
- avoids inventing facts that are not in the user profile or product data.

An LLM judge is more flexible, but it introduces another consistency problem. The FAQ suggests calibrating the judge with a small human-annotated dataset. Otherwise, GEPA may optimize for the judge's habits instead of the behavior I actually want.

#### A hybrid evaluator

For my agent, a hybrid evaluator seems more realistic than choosing only one approach.

I can use deterministic code for non-negotiable conditions such as valid output, existing product IDs, required fields, and token limits. I can then use an LLM judge for preference quality and explanation quality. The evaluator can combine those results into a score and return detailed feedback about each failed condition.

The feedback matters as much as the score. A result like `0.4` only tells GEPA that something went wrong. Feedback such as the following tells the reflection model what it may need to fix:

```text
The response used valid product IDs and stayed within the token limit.
However, it ranked items by price alone and ignored the user's preference history.
The explanation also mentioned a discount that was not present in the input data.
```

GEPA calls this diagnostic context Actionable Side Information, or ASI. It can include the expected and actual outputs, error messages, constraint violations, domain hints, and separate objective scores. The official FAQ recommends returning this information instead of a score alone.

### 7. Let the reflection model propose a new prompt

The reflection model receives the selected prompt and the minibatch records from the previous step. Those records include the inputs, generated outputs, and feedback.

It then proposes a revised instruction. GEPA copies the parent and replaces only the selected prompt. Any other prompts remain unchanged.

I initially thought the reflection model might combine every prompt in the pool. It does not. Standard reflective mutation starts from the single parent selected by Algorithm 2. GEPA+Merge is a separate variant that combines changes from two lineages.

### 8. Compare the parent and the child

The new candidate is evaluated on the same feedback minibatch that was used to evaluate its parent. The default acceptance rule compares the sum of the per-example scores.

The child must be strictly better than the parent. If the score is lower, it is rejected. If the score is equal, it is also rejected under the default `strict_improvement` setting.

If the child improves, the paper's algorithm adds it to the candidate pool and evaluates it on all examples in `D_pareto`. Its per-example validation scores then become available to Algorithm 2 in later iterations.

The process continues until the rollout budget is exhausted. GEPA finally returns the single candidate with the best average score on `D_pareto`. Pareto selection is used to maintain diversity while searching; the final output is still one candidate.

### The complete loop

My original understanding was close, but the corrected order is:

1. Split the data into a feedback set and a Pareto selection set.
2. Evaluate the seed candidate on the full Pareto set.
3. For every Pareto example, find all candidates tied for the best score.
4. Remove a candidate only when every case it wins still has at least one other surviving winner.
5. Sample one parent in proportion to the number of cases it wins.
6. Select one prompt to update. If there are several, use round-robin by default.
7. Sample a random feedback minibatch and run the parent.
8. Score the outputs with a custom evaluator and collect diagnostic feedback.
9. Ask the reflection model to revise the selected prompt.
10. Run the child on the same minibatch.
11. Reject the child unless it strictly improves over the parent.
12. Evaluate an accepted child on the full Pareto set and repeat.

The most important correction is that steps 3-5 choose a parent. Step 11 controls admission to the candidate pool.

## Conclusion

### Applying this to my recommendation agent

The official FAQ lists model migration as a GEPA use case: keep the program structure, switch the task model, and optimize the prompt again. That is close to my situation, but my recommendation agent has several competing requirements.

Suppose a user asks for inexpensive products. Price should not be the only signal. The agent may also need to consider their usual food choices and discount history, then explain the trade-off. GEPA cannot discover that policy from a vague success score. I need to make it visible in the dataset and evaluator.

A useful example may need to contain:

- the user's request;
- the preference information the agent is allowed to use;
- the available products and discounts;
- the hard constraints that must not be violated;
- an expected ranking or a human-written assessment;
- an explanation of why the expected behavior is correct.

Code could reject unavailable products or malformed output. An LLM judge could assess whether the ranking balances the request with the user's preferences and whether the explanation stays grounded in the data.

GEPA can optimize against those signals. It cannot remove the need to define them.

### Choosing a seed without losing token savings

I first considered two seeds. One was the existing prompt, including parts tuned for the previous model. The other was a minimal prompt that described the job and response format with almost no modifiers. I sometimes call this a "caveman" style because it uses as few tokens as possible.

The [FAQ's guidance on seed prompts](https://gepa-ai.github.io/gepa/guides/faq/#how-does-the-seed-prompt-affect-optimization) gave me a better boundary. A good seed contains the ground rules and constraints a smart human would need. It leaves out process details that the model should discover through practice.

I have started tuning again with that boundary. I keep the business rules, tool constraints, and output contract. I leave the exact procedure and wording open to optimization. I also keep the original prompt as a baseline. Model-specific reminders and repeated instructions are candidates for removal, but only after testing.

The [official FAQ is direct about prompt length](https://gepa-ai.github.io/gepa/guides/faq/#does-gepa-aim-for-brevity-in-prompts). GEPA does not optimize for brevity unless brevity is part of the objective. Early candidates can be long. They may also copy details from the first training examples they see.

Later iterations may produce more general rules, but that does not replace a token constraint. The documentation suggests several controls:

- add prompt length or token cost as an evaluation objective;
- add an explicit maximum length to the reflection prompt;
- tell the proposer not to copy example-specific names or phrases;
- preserve fixed output schemas and task rules through explicit constraints;
- ask for the smallest targeted edit instead of a complete rewrite.

For a production prompt, I prefer the smallest reviewable edit. The FAQ also suggests a human-written instruction library, with a custom proposal function selecting rules instead of generating unrestricted text.

### The dataset is still the hard part

The [official FAQ says GEPA can work with very few examples](https://gepa-ai.github.io/gepa/guides/faq/#can-gepa-work-with-very-few-examples) and recommends roughly 30 to 300 for better results. My problem is not only the count. The data must cover low prices, strong or sparse preferences, conflicting signals, unavailable discounts, and explanation quality. The validation set must also represent real traffic.

For a tool-using agent, a dataset entry is not just a user question and a final answer. The final response may depend on several intermediate steps:

```text
User request
-> user context available to the agent
-> tool name and arguments
-> tool message and tool response
-> ranking or decision made from that response
-> final answer
-> evaluation feedback
```

The [official guidance on agent traces](https://gepa-ai.github.io/gepa/guides/faq/#how-should-i-serialize-agent-traces-for-gepa) recommends recording relevant state changes, decisions, outputs, and failures. I therefore need examples of both the final answer and the steps that produced it. Those examples should cover empty tool responses, tool errors, mismatched products, conflicting preferences, and misleading discounts.

The feedback must also locate the failure. If the tool returned the right products but the answer ranked them incorrectly, the reflection model should not change the tool call. If the tool request was wrong, writing feedback will not fix it. The full trajectory makes this distinction possible.

This brought back a lesson from my year-long undergraduate internship in a computer vision lab. It was easy to focus on the model and training code, but dataset quality often decided what the model could learn and whether an evaluation meant anything. GEPA changes prompts instead of weights, but it still learns only from the examples and feedback I provide.

### My first experiment

I plan to start with a narrow experiment rather than optimize the whole agent at once.

1. Keep the current agent flow and use the smaller task model.
2. Preserve the task rules, tool contracts, and output format in the seed. Keep the original prompt as a baseline.
3. Start with one prompt and a small set of recommendation scenarios.
4. Capture the complete tool trajectory.
5. Use code for hard constraints and an LLM judge for recommendation quality.
6. Track prompt length and keep separate feedback, validation, and test data.
7. Review every accepted prompt and check the final result on untouched test and regression cases.

This will not prove that GEPA works for the entire agent. It will show whether my evaluation signal can improve one part of it.

### What I learned

GEPA became easier to understand once I separated two decisions. Pareto selection chooses the parent. The minibatch comparison decides whether its child enters the pool.

For my agent, the mutation algorithm is not the hardest part. I first need realistic examples and feedback that identifies which step failed. I also need to measure recommendation quality and token cost without rewarding the wrong behavior.

I am still tuning the prompt, and I remain skeptical about whether the result will be reliable enough to adopt. I cannot claim that GEPA works for my agent yet. This does not contradict the paper, which uses defined benchmarks and metrics. It means that, in my case, the immediate bottleneck is the dataset and evaluator.

### References

- [GEPA paper](https://arxiv.org/abs/2507.19457)
- [GEPA official repository](https://github.com/gepa-ai/gepa)
- [GEPA paper artifact](https://github.com/gepa-ai/gepa-artifact)
- [GEPA FAQ](https://gepa-ai.github.io/gepa/guides/faq/)
- [Creating GEPA adapters](https://gepa-ai.github.io/gepa/guides/adapters/)
- [Default evaluator implementation](https://github.com/gepa-ai/gepa/blob/b265bf9ca77fd8e8d82039d9f74911b8780fe1ce/src/gepa/adapters/default_adapter/default_adapter.py#L49-L74)
- [Strict improvement acceptance rule](https://github.com/gepa-ai/gepa/blob/b265bf9ca77fd8e8d82039d9f74911b8780fe1ce/src/gepa/strategies/acceptance.py#L44-L53)
