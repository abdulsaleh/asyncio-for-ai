# Introduction

If you've used the OpenAI or Gemini API, you've probably seen `asyncio` throughout the documentation. You might've copied `async/await` without understanding what it does. You might've wondered if it actually matters.

It matters.

`asyncio` is a library for writing concurrent code in Python. You can use `asyncio` to make concurrent model API calls, sending 100s of requests concurrently instead of one at a time.

Making API calls is a common use case of `asyncio`, but you can also use it to build way cooler things like rate limiters, producer-consumer pipelines, and web crawlers. This guide is about building those cool things.


## How this guide is structured

This guide walks through six real-world applications of `asyncio`:

* **[LLM Responses](llm-responses.md)** — Call LLM APIs concurrently without blocking
* **[Rate Limiters](rate-limiters.md)** — Control API request throughput to stay within rate limits
* **[Data Pipelines](data-pipelines.md)** — Process large datasets with producer-consumer pipelines
* **[Request Batchers](request-batchers.md)** — Batch API requests for efficiency
* **[Web Crawlers](web-crawlers.md)** — Efficiently crawl the web and parse web pages
* **[Function-Calling Agents](function-agents.md)** — Build agents that execute tools concurrently

Each section follows a challenge-solution format inspired by John Crickett's [Coding Challenges](https://codingchallenges.substack.com/).

Try to solve each challenge yourself first. Research shows that [productive failure](https://pubmed.ncbi.nlm.nih.gov/33180211/) leads to deeper learning, even if you struggle and can't come up with the solution.

## About me

Hi! I'm Abdul. I build infrastructure for Gemini fine-tuning and batch inference at Google. I care about making AI development easier, faster, and more accessible.

Check out my blog at [abdulsaleh.dev](https://www.abdulsaleh.dev/).

## Source Code

You can find the markdown source code for this guide on [github](https://github.com/abdulsaleh/asyncio-for-ai). It was compiled in Rust using [mdbook](https://rust-lang.github.io/mdBook/).
