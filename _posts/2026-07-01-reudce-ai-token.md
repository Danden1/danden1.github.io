---
title: "How to Reduce Token Usage When Developing AI Agents"
description: "What I learned while reducing token usage in an AI agent built with LangChain."
date: 2026-07-01 21:29:00 +0900
categories: [AI, Agent]
tags: [Langchain, AI, Agent, Token]
---

Currently, I am developing AI agents at work using LangChain.

I cannot share the internal details of the project, but I can share what I learned while reducing token usage.

When building an AI agent, token usage is not just a technical detail. It directly affects cost, latency, and even the business model. If every user request consumes too many tokens, the service can become too expensive to operate. This is especially important when the agent is provided to real customers.

In this post, I will explain the methods I used or considered to reduce token usage:

1. Prompt caching
2. Tool design
3. Compact data formats
4. Reducing output tokens

The most important lesson for me was simple:

> Do not just optimize the prompt. Optimize the whole agent flow.

## Prompt Caching

Prompt caching is one of the easiest ways to reduce input token costs and latency.

In many agents, the beginning of the prompt is almost always the same. For example:

- System prompt
- Agent instructions
- Tool names
- Tool descriptions
- Tool input schemas

These parts are often sent to the model again and again. If the model provider can reuse the same prefix, the repeated part can be treated as cached input.

OpenAI documents that prompt caching is available for prompts containing 1,024 tokens or more, and the API response includes a `cached_tokens` field under `usage.prompt_tokens_details`. See the official documentation: [OpenAI Prompt Caching](https://developers.openai.com/api/docs/guides/prompt-caching).

This means prompt caching is especially useful when the agent has a long system prompt or many tools.

However, making the system prompt too short is not always the best choice. If I reduce the stable prompt too aggressively, the request may not reach the provider's caching threshold, or the reusable prefix may become too small to provide much benefit. In that case, the same instructions can be billed as normal input tokens on every request. A slightly longer but cache-friendly system prompt can sometimes be cheaper than a very short prompt that cannot benefit from caching.

Prompt structure also matters. In practice, the stable parts should stay near the beginning of the prompt. Dynamic values should be placed later.

For example, this is cache-friendly:

```text
System prompt
Tool descriptions
Stable policy
User message
Current time
Request-specific data
```

This is less cache-friendly:

```text
System prompt
Current time
Tool descriptions
Stable policy
User message
```

If a dynamic value such as the current time is placed near the top, it can change on every request. That makes the prefix different, so the provider may not be able to reuse the cached part after that point.

### OpenAI and Anthropic

Provider behavior varies.

For OpenAI models, prompt caching is handled automatically when the request meets the caching requirements. I still need to design the prompt carefully so that the stable prefix can be reused.

For Anthropic models, prompt caching can be controlled more explicitly with `cache_control`. Anthropic also documents different cache durations, including a 5-minute cache and a 1-hour cache with additional cost. See the official documentation: [Anthropic Prompt Caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching).

The important point is that prompt caching is not free magic. Each provider has different rules, TTLs, and pricing. Before relying on it, I need to check the model provider's documentation and pricing page.

## Reducing LLM Calls Through Tool Design

The next optimization—reducing the number of LLM calls—is more important than prompt caching.

In an agent, a single user request can cause multiple model calls. This happens because the model may first decide which tool to call, then the tool runs, and then the result is sent back to the model.

LangChain's reference documentation says that `create_agent` creates an agent graph that calls tools in a loop until a stopping condition is met. It also explains the flow: the agent node calls the language model, tool calls are executed, tool responses are added as `ToolMessage` objects, and the agent node calls the language model again until there are no more tool calls. See: [LangChain `create_agent` reference](https://reference.langchain.com/python/langchain/agents/#langchain.agents.create_agent).

This means tool design can directly affect cost.

### A Simple LangChain Tool Example

LangChain supports defining tools with the `@tool` decorator. The official documentation says the function docstring becomes the tool description, and type hints define the tool input schema. See: [LangChain Tools documentation](https://docs.langchain.com/oss/python/langchain/tools).

Here is a simple example using the LangChain 1.x style:

```python
from langchain.agents import create_agent
from langchain.tools import tool


@tool
def get_weather(city: str) -> str:
    """Get the current weather for a city."""
    return f"The weather in {city} is sunny."


agent = create_agent(
    model="openai:gpt-5.4-mini",
    tools=[get_weather],
    system_prompt="You are a helpful assistant. Be concise and accurate.",
)

result = agent.invoke(
    {"messages": [{"role": "user", "content": "What is the weather in Seoul?"}]}
)
```

This is a simple example, but it shows the main idea: the model receives the tool name, description, and schema, then decides whether to call it.

### Tool Design Mistake: Too Many Small Tools

When I first thought about tool design, I was tempted to separate every small action into its own tool.

For example, imagine a recommendation agent.

One possible design is:

1. The user asks for a recommendation.
2. The LLM decides to call `get_user_preferences`.
3. The tool returns the user's preferences.
4. The LLM reads the preferences and decides to call `recommend_items`.
5. The recommendation tool returns candidate items.
6. The LLM writes the final answer.

This design is easy to understand, but it can require more model calls.

In many cases, a better design is to make one domain-level tool:

```python
from langchain.tools import tool


@tool
def recommend_items(user_id: str, user_message: str, limit: int = 5) -> str:
    """Recommend items for a user based on the user's request and stored preferences."""
    preferences = load_user_preferences(user_id)
    items = search_recommendations(
        preferences=preferences,
        user_message=user_message,
        limit=limit,
    )
    return format_recommendation_result(items)
```

In this design, the tool itself loads the user's preferences and performs the recommendation logic. The LLM does not need to plan every internal step.

This does not mean every tool should become large. The point is to avoid exposing internal application steps as separate LLM decisions when normal code can handle them more cheaply and more reliably.

My rule of thumb is:

> If the step is deterministic application logic, keep it in code. If the step requires language understanding or judgment, let the LLM handle it.

This helped reduce both token usage and response time.

## Data Format

Another useful method is to change the data format returned by tools.

JSON is easy to use, but it can contain many repeated field names.

For example:

```json
{
  "items": [
    {
      "id": 1,
      "name": "one",
      "price": 100
    },
    {
      "id": 2,
      "name": "two",
      "price": 10
    }
  ]
}
```

In this small example, the field names `id`, `name`, and `price` are repeated for every item. With a small response, this is not a big issue. But with many rows, repeated field names can increase the number of input tokens.

Depending on the domain, a more compact format may be better.

For example, the same data can be represented as CSV:

```csv
id,name,price
1,one,100
2,two,10
```

Or as a compact text format:

```text
id|name|price
1|one|100
2|two|10
```

The best format depends on the task.

JSON is still a good default when the structure is complex or when the model must understand nested data. But if the data is table-like, CSV or another compact format can be easier and cheaper for the model to read.

When designing a tool, I now ask:

- Does the LLM really need full JSON?
- Is the data table-like?
- Can I remove fields that are not needed?
- Can I return IDs instead of full objects?
- Can I summarize data before sending it to the model?

The same idea also applies to model output. If the application only needs a small structured result, I should not ask the model to write a long answer.

## Reducing Output Tokens

Input tokens matter, but output tokens are often more expensive.

As of July 1, 2026, OpenAI's pricing page lists `gpt-5.4-mini` standard short-context pricing as `$0.375` per 1M input tokens and `$2.25` per 1M output tokens. That makes output tokens about 6 times more expensive than input tokens for that model. See: [OpenAI API Pricing](https://developers.openai.com/api/docs/pricing).

Because of this, reducing output tokens can be very effective.

One practical method is to make the LLM return only the data that requires language understanding.

For example, suppose the application has product data in a database. A simple but expensive approach is:

1. Send product details to the LLM.
2. Ask the LLM to choose products.
3. Ask the LLM to return full product names, descriptions, prices, and explanations.

This can create many output tokens.

A cheaper approach is:

1. Send only the necessary product summary to the LLM.
2. Ask the LLM to return selected product IDs.
3. Fetch the full product details in application code.
4. Render the final UI or response with normal code.

For example:

```json
{
  "selected_item_ids": [101, 205, 309]
}
```

This is much shorter than asking the model to generate full item objects.

In my case, I used this approach and reduced token cost by more than 35%.

The key idea is:

> Let the LLM make the decision, but let the application build the final response.

This is also easier to maintain. The application already knows how to fetch database records, apply business rules, and format the final response. The LLM does not need to regenerate information that already exists in the system.

This approach can also reduce hallucination risk. If the model only returns IDs, the application can validate whether those IDs exist before showing anything to the user.

## What I Learned

Before working on this, I mainly thought about prompt optimization.

But while developing an agent, I realized that token optimization is more like system design.

Prompt caching helps, but it is only one part of the problem. Tool design, data format, output format, and application architecture are also important.

The most useful questions were:

- Can this step be handled by code instead of the LLM?
- Can I reduce the number of tool calls?
- Can I move repeated information into a cached prefix?
- Can I return IDs instead of full objects?
- Can I use a compact data format?
- Can I make the model output only the minimum required result?

These questions helped me think about the agent as a production system, not just a prompt.

## Conclusion

AI agents can be useful, but their token usage matters.

Building a custom LLM pipeline might allow more aggressive optimization of cost and latency. In many real projects, however, developers use model APIs and agent frameworks. In that environment, decisions about prompts, tools, and data flow have a large impact.

This is even more important when the service is offered to customers. A poorly designed agent can become expensive very quickly. In the worst case, the business can lose money every time users interact with the agent.

While working on this, I also became more impressed by GPT and Claude. They handle large contexts, many users, and complex instructions remarkably well. At the same time, I became curious about how large AI products manage context, cost, and quality internally.

I am not an AI researcher. There may be more advanced methods, such as reinforcement learning or supervised fine-tuning, depending on the problem. At the MS AI Tour I attended earlier this year, I heard that only a small number of teams were using RL or SFT in production. That made me think that many teams may still be solving these problems with engineering design first.

For now, my biggest takeaway is this:

> Token reduction is not just a prompt engineering problem. It is also an application design problem.
