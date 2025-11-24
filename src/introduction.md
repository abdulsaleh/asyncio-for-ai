# Introduction

AI applications spend a lot of time waiting.

They wait on model responses, database queries, or even agent tool calls. Most AI applications are IO-bound. They wait on external services or disk reads, not CPU cycles.

`asyncio` is a library for writing concurrent code with `async/await` syntax in Python. It can help you write faster, more scalable code by doing IO work concurrently. For example, instead of sending one model request at a time, you can send many at once.

Making concurrent API calls is the most common use of `asyncio`, but you can build way cooler things like rate limiters, producer-consumer pipelines, or web crawlers. This is a guide about building those cool things.

## How this guide is structured

This guide walks through six real-world applications of `asyncio`:

* **[LLM Responses](llm-responses.md)** — Call LLM APIs concurrently without blocking
* **[Rate Limiters](rate-limiters.md)** — Control API request throughput to stay within rate limits
* **[Data Pipelines](data-pipelines.md)** — Process large datasets with producer-consumer pipelines
* **Request Batchers** — Batch API requests for efficiency
* **Web Crawlers** — Efficiently crawl the web and parse web pages
* **Tool-Calling Agents** — Build agents that execute tools concurrently

Each section follows a challenge-solution format inspired by John Crickett's [Coding Challenges](https://codingchallenges.substack.com/).

Try to solve each challenge yourself first. Research shows that [productive failure](https://pubmed.ncbi.nlm.nih.gov/33180211/) leads to deeper learning, even if you struggle and can't come up with the solution.

## About me

Hi! I'm Abdul. I build infrastructure for Gemini fine-tuning and batch inference at Google. I care about making AI development easier, faster, and more accessible.

If you find this guide helpful, check out my blog at [abdulsaleh.dev](https://www.abdulsaleh.dev/).

## Source Code

You can find the markdown source code for this guide on [github](https://github.com/abdulsaleh/asyncio-for-ai). It was compiled in Rust using [mdbook](https://rust-lang.github.io/mdBook/).
