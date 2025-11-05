# Introduction

If you've used the OpenAI or Gemini API, you've likely seen `asyncio` and `async/await` scattered throughout example code. Maybe you copied the syntax without fully understanding it. Maybe you've wondered if it actually matters.

It matters.

Most `asyncio` tutorials talk about concurrency primitives, event loops, and coroutines. This guide focuses on why it matters and what you can build with it. If you're building AI or LLM applications, `asyncio` is important to writing fast, scalable systems.

From data pipelines that process thousands of documents in parallel to agents that handle multiple tool calls simultaneously, concurrency is everywhere in modern AI development. This guide will show you how to harness it. 


## What you'll build

This guide walks through seven real-world applications of `asyncio` in action:

* **LLM Responses** — Call LLM APIs concurrently without blocking
* **Rate Limiters** — Control API request throughput to stay within rate limits
* **Request Batchers** — Combine multiple requests for efficiency
* **Data Pipelines** — Process large datasets concurrently
* **Web Crawlers** — Crawl and parse web pages in parallel
* **Tool-Calling Agents** — Build agents that execute tools concurrently
* **Parallel Agents** — Run multiple independent agents simultaneously

Each section follows a challenge-solution format inspired by John Crickett's [Coding Challenges](https://codingchallenges.substack.com/). You'll see the problem first, then walk through a complete implementation.

I encourage you to attempt each challenge before reading the solution. Research shows that [productive failure](https://pubmed.ncbi.nlm.nih.gov/33180211/) leads to deeper learning—even if you don't solve it perfectly, the struggle makes the solution more memorable.


## About me

I'm a software engineer at Google, where I build infrastructure for Gemini fine-tuning and batch inference. I care about making AI development easier, faster, and more accessible. 

If you find this guide helpful, check out my at [abdulsaleh.dev](https://www.abdulsaleh.dev/).
