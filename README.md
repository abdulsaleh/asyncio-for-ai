# AsyncIO for AI Applications

A practical guide to using Python's `asyncio` for building faster, more scalable LLM and AI applications.

## Read the Guide

**[Read online at asyncio-for-ai.github.io](https://abdulsaleh.github.io/asyncio-for-ai/)**

## About This Guide

If you've used the OpenAI or Gemini API, you've probably seen `asyncio` scattered throughout the documentation. Maybe you've copied `async/await` without understanding what it does. Maybe you've wondered if it actually matters.

It matters.

`asyncio` is a library for writing concurrent code in Python. It can help you write faster, more scalable code, whether you're making API calls, building data pipelines, or orchestrating agents.

Most `asyncio` tutorials explain how it works, focusing on concurrency primitives. **This guide is about why it matters.**

You’ll learn how to use `asyncio` to build faster, more scalable systems.

## What You'll Learn

This guide walks through seven real-world applications of `asyncio`:

- **LLM Responses** — Call LLM APIs concurrently without blocking
- **Rate Limiters** — Control API request throughput to stay within rate limits
- **Request Batchers** — Batch multiple requests for efficiency
- **Data Pipelines** — Process large datasets with producer-consumer pipelines
- **Web Crawlers** — Efficiently crawl the web and parse web pages
- **Tool-Calling Agents** — Build agents that execute tools concurrently
- **Parallel Agents** — Run multiple independent agents simultaneously

Each section follows a challenge-solution format. You're encouraged to attempt each challenge before reading the solution.

## About me

I'm a software engineer at Google, where I build infrastructure for Gemini fine-tuning and batch inference. I care about making AI development easier, faster, and more accessible.

Check out my blog at [abdulsaleh.dev](https://www.abdulsaleh.dev/).

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
