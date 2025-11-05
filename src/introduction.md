# Introduction

If you've ever used model APIs like the OpenAI API, you've probably encountered `asyncio` and `async/await` syntax.

There are many `asyncio` guides out there, but most focus on concurrency primitives, coroutines, event loops, and other low-level details.

This guide takes a different approach: learning through building cool stuff, especially if you're interested in AI and LLM applications.

`asyncio` makes it easy to write concurrent code in Python. Concurrency powers many AI applications, from efficient data pipelines and web crawlers to interactive tool-calling agents. 


## How this guide is structured

This guide covers six real-world applications of `asyncio` relevant to AI:

* LLM Responses
* Rate Limiters
* Request Batchers
* Data Pipelines
* Web Crawlers
* Tool-Calling Agents
* Parallel Agents

Inspired by John Crickett's [Coding Challenges](https://codingchallenges.substack.com/), each section is split into a "challenge" that describes the problem and a "solution" that walks through the implementation.

Try to tackle each challenge without looking at the solution first. Even if you can't solve it initially, you'll learn more from [productive failure](https://pubmed.ncbi.nlm.nih.gov/33180211/).


## About me

I'm a software engineer at Google, where I build infrastructure for Gemini fine-tuning
and batch inference. I care about building tools that make AI development easier, more
efficient, and more accessible.

Check out my [blog](https://www.abdulsaleh.dev/) for other guides I've written.
