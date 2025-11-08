# Introduction

If you've used the OpenAI or Gemini API, you've probably seen `asyncio` scattered throughout the code examples. Maybe you copied `async/await` without understanding what it does. Maybe you've wondered if it actually matters.

It matters.

If you're building AI applications, `asyncio`  can keep your code fast and responsive, whether you're making hundreds of API calls, streaming data through a pipeline, or coordinating agent tool calls.

Most `asyncio` tutorials explain *how* it works, focusing on concurrency primitives, event loops, and coroutines. **This guide is about why it matters.**

You’ll learn how to use `asyncio` to build faster, more scalable systems.

## How this guide is structured

This guide walks through seven real-world applications of `asyncio`:

* **[LLM Responses](llm-responses.md)** — Call LLM APIs concurrently without blocking
* **[Rate Limiters](rate-limiters.md)** — Control API request throughput to stay within rate limits
* **[Request Batchers](request-batchers.md)** — Batch multiple requests for efficiency
* **[Data Pipelines](data-pipelines.md)** — Process large datasets with producer-consumer pipelines
* **[Web Crawlers](web-crawlers.md)** — Efficiently crawl the web and parse web pages
* **[Tool-Calling Agents](tool-agents.md)** — Build agents that execute tools concurrently
* **[Parallel Agents](parallel-agents.md)** — Run multiple independent agents simultaneously

Each section follows a challenge-solution format inspired by John Crickett's [Coding Challenges](https://codingchallenges.substack.com/).

Try to solve each challenge yourself first. Research shows that [productive failure](https://pubmed.ncbi.nlm.nih.gov/33180211/) leads to deeper learning, even if you struggle and can't come up with the solution.

## About me

I'm a software engineer at Google, where I build infrastructure for Gemini fine-tuning and batch inference. I care about making AI development easier, faster, and more accessible.

If you find this guide helpful, check out my blog at [abdulsaleh.dev](https://www.abdulsaleh.dev/).

## Source Code

You can find the markdown source code for this guide on [github](https://github.com/abdulsaleh/asyncio-for-ai). This book was compiled in Rust using [mdbook](https://rust-lang.github.io/mdBook/).