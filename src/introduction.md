# Introduction

If you've used the OpenAI or Gemini API, you've likely seen `asyncio` scattered throughout the code examples. Maybe you copied `async/await` without understanding what it does. Maybe you've wondered if it actually matters.

It matters.

Most `asyncio` tutorials talk about concurrency primitives, event loops, and coroutines. **This guide is different**. It focuses on why `asyncio` matters and what you can build with it. 

If you're building AI or LLM applications, `asyncio` can help you write faster more scalable code.

<!-- AI and LLM applications have many opportunities for concurrency: calling multiple LLM APIs concurrently, processing hundreds of documents in a pipeline, or executing agent tool calls simultaneously. Your code doesn't need to wait around doing nothing. This guide will show you how to harness it. -->

## How this guide is structured

This guide walks through seven real-world applications of `asyncio`:

* **[LLM Responses](llm-responses.md)** — Call LLM APIs concurrently without blocking
* **[Rate Limiters](rate-limiters.md)** — Control API request throughput to stay within rate limits
* **[Request Batchers](request-batchers.md)** — Combine multiple requests for efficiency
* **[Data Pipelines](data-pipelines.md)** — Process large datasets concurrently
* **[Web Crawlers](web-crawlers.md)** — Crawl and parse web pages in parallel
* **[Tool-Calling Agents](tool-agents.md)** — Build agents that execute tools concurrently
* **[Parallel Agents](parallel-agents.md)** — Run multiple independent agents simultaneously

Each section follows a challenge-solution format inspired by John Crickett's [Coding Challenges](https://codingchallenges.substack.com/).

I encourage you to attempt each challenge before reading the solution. Research shows that [productive failure](https://pubmed.ncbi.nlm.nih.gov/33180211/) leads to deeper learning, even if you struggle and can't come up with the solution.


## About me

I'm a software engineer at Google, where I build infrastructure for Gemini fine-tuning and batch inference. I care about making AI development easier, faster, and more accessible.

If you find this guide helpful, check out my at [abdulsaleh.dev](https://www.abdulsaleh.dev/).
