---
title: Agentic Context Engineering
description: How to maintain context for AI agent
date: 2025-12-25 23:37:00 +0900
categories: [AI, Agent]
tags: [AI Agent, Paper, Context]
math: true
image: "/assets/img/posts/ace/image2.png"
# mermaid: true
---

Currently, I work on developing AI agents at my company.

Although my primary role is as a backend developer, I am interested in AI and in improving overall team productivity.

My goal is to use AI agents to answer questions from other team members and to assist with server operations.

I have built an AI agent using LangChain, but I encountered a limitation: the agent does not retain previous conversation context. While LangChain provides features such as context summarization and memory modules, this led me to question how context should be designed and managed more effectively.

As a result, I began researching context engineering.

And I found an interesting paper: Agentic Context Engineering.

## Background

### Context Adaptation
Context adaptation (or context engineering) refers to methods that improve model behavior by constructing
or modifying inputs to an LLM, rather than altering its weights

## Agentic Context Engineering (ACE)

### Introduction

Context engineering can reduce hallucination.

Exsisting approaches to context adaption face to key limitations.

**Brevity Bias** : the tendency for instructions to collapse into short, generic commands like "Do a good job." While they look clean, they lack the domain-specific nuances required for complex tasks. This leads to a loss of critical information and causes the model to repeat the same errors.

**Context Collapse** : As the context grows large, the model tends to compress it into much hsorter, less informative sumeries.

![Context Collapse](/assets/img/posts/ace/image.png)
_source: [ACE](https://arxiv.org/pdf/2510.04618)_

This paper argue that contexts should function not as concise summaries, but as comprehensive, evolving playbooks—detailed, inclusive, and rich with domain insights.

### Architecture of ACE

At the heart of ACE is the "Playbook"—a dynamic collection of strategies, tips, and lessons learned that grows more effective over time.

![Arch](/assets/img/posts/ace/image2.png)
_source: [ACE](https://arxiv.org/pdf/2510.04618)_


ACE divides context adaptation into three modular roles — Generator, Reflector, and Curator — that together build a structured, incrementally evolving playbook.

The **Generator** executes reasoning paths and exposes helpful vs harmful patterns, the **Reflector** distills actionable lessons, and the **Curator** merges these into the global context using deterministic, non-LLM logic (e.g., semantic deduplication, pruning, delta updates) to avoid context collapse and reduce cost. Curator’s algorithmic merge.

- **Incremental Delta Updates** : Update only minimal, identifiable units rather than the full context, which reduces context drift and improves stability.

- **Grow-and-Refine** : Continuously accumulate knowledge while periodically pruning noise and redundancy to maintain the quality and compactness of the playbook.


After Run, Playbook is made.

```json
{
  "skills": {
    "web_interaction_patterns-00001": {
      "id": "web_interaction_patterns-00001",
      "section": "web_interaction_patterns",
      "content": "Use shadowRoot.querySelector() for elements hidden in shadow DOM",
      "helpful": 2,
      "harmful": 0,
      "neutral": 0,
      "created_at": "2025-11-24T23:27:19.739470+00:00",
      "updated_at": "2025-11-24T23:27:19.739486+00:00"
    },
    "web_interaction_patterns-00002": {
      "id": "web_interaction_patterns-00002",
      "section": "web_interaction_patterns",
      "content": "Implement getAllShadowRoots() recursive traversal when querySelector fails",
      "helpful": 0,
      "harmful": 0,
      "neutral": 0,
      "created_at": "2025-11-24T23:27:19.739513+00:00",
      "updated_at": "2025-11-24T23:27:19.739518+00:00"
    }
  }
}
```

_source: [github](https://github.com/kayba-ai/agentic-context-engine)_

### Results

![Results](/assets/img/posts/ace/image3.png)
_source: [ACE](https://arxiv.org/pdf/2510.04618)_

I think that using LLMs for all three roles—Generator, Reflector, and Curator—would result in high operational costs.

However, the actual outcome was different.

#### Code

[github](https://github.com/kayba-ai/agentic-context-engine)

I implemented this using LangChain, but it does not work properly with LangChain v1.0 or later (2025-12-24).

The reason is that the following classes and methods have been deprecated in LangChain v1.x:
```python
from langchain.agents import AgentExecutor, create_tool_calling_agent
```

Related issue : [github issue](https://github.com/kayba-ai/agentic-context-engine/issues/45)