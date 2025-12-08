# asyncio for AI Applications

A practical guide to using Python's `asyncio` for building faster, more scalable LLM and AI applications.

## Read the Guide

**[Read online at asyncio-for-ai.github.io](https://abdulsaleh.github.io/asyncio-for-ai/)**

## About This Guide

If you've used the OpenAI or Gemini API, you've probably seen `asyncio` throughout the documentation. You might've copied `async/await` without understanding what it does. You might've wondered if it actually matters.

It matters.

`asyncio` is a library for writing concurrent code in Python. You can use `asyncio` to make concurrent model API calls, sending 100s of requests concurrently instead of one at a time.

Making API calls is a common use case of `asyncio`, but you can also use it to build way cooler things like rate limiters, producer-consumer pipelines, and web crawlers. This guide is about building those cool things.


## How this guide is structured

This guide walks through six real-world applications of `asyncio`:

* **LLM Responses** — Call LLM APIs concurrently without blocking
* **Rate Limiters** — Control API request throughput to stay within rate limits
* **Data Pipelines** — Process large datasets with producer-consumer pipelines
* **Request Batchers** — Batch API requests for efficiency
* **Web Crawlers** — Efficiently crawl the web and parse web pages
* **Function-Calling Agents** — Build agents that execute tools concurrently

Each section follows a challenge-solution format inspired by John Crickett's [Coding Challenges](https://codingchallenges.substack.com/).

Try to solve each challenge yourself first. Research shows that [productive failure](https://pubmed.ncbi.nlm.nih.gov/33180211/) leads to deeper learning, even if you struggle and can't come up with the solution.

## About me

Hi! I'm Abdul. I build infrastructure for Gemini fine-tuning and batch inference at Google. I care about making AI development easier, faster, and more accessible.

If you find this guide helpful, check out my blog at [abdulsaleh.dev](https://www.abdulsaleh.dev/).

## Building Locally

This book is built using [mdBook](https://rust-lang.github.io/mdBook/).

```bash
# Install mdBook
cargo install mdbook

# Build and serve locally
mdbook serve

# Build static site
mdbook build
```

The book will be available at `http://localhost:3000`.

## Contributing

Found a typo or have a suggestion? Feel free to open an issue or submit a pull request!
