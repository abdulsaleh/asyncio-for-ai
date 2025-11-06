# AsyncIO for AI Applications

A practical guide to using Python's `asyncio` for building faster, more scalable AI and LLM applications.

## Read the Book

**[Read online at asyncio-for-ai.github.io](https://asyncio-for-ai.github.io)**

## About This Book

If you've used the OpenAI or Gemini API, you've likely seen `asyncio` scattered throughout the code examples. Maybe you copied `async/await` without understanding what it does. Maybe you've wondered if it actually matters.

It matters.

Most `asyncio` tutorials talk about concurrency primitives, event loops, and coroutines. **This guide is different**. It focuses on why `asyncio` matters and what you can build with it.

## What You'll Learn

This guide walks through seven real-world applications of `asyncio`:

- **LLM Responses** — Call LLM APIs concurrently without blocking
- **Rate Limiters** — Control API request throughput to stay within rate limits
- **Request Batchers** — Combine multiple requests for efficiency
- **Data Pipelines** — Process large datasets concurrently
- **Web Crawlers** — Crawl and parse web pages in parallel
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

## License

All content is available for educational purposes.
