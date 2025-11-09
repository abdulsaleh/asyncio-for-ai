# Async LLM Responses

## Challenge

When working with LLMs, you often need to make multiple API calls to generate responses for different prompts or handle multiple user requests. Making these calls synchronously means waiting for each response before sending the next request, which is slow and inefficient.

In this challenge, you'll use `asyncio` to send multiple concurrent requests to the Gemini API. You'll start with a basic synchronous implementation, measure its performance, then use `asyncio` to speed up execution.

### Step 0

To get started, get a Gemini API key from [Google AI Studio](https://aistudio.google.com/app/api-keys). We use the Gemini API because it has a generous free tier.

```bash
export GEMINI_API_KEY="YOUR_API_KEY"
```

### Step 1

In this step, your goal is to verify you can call the Gemini API and get responses.

Install the Google [GenAI SDK](https://ai.google.dev/gemini-api/docs/quickstart) and make your first request. Write your code to `script.py`.

```bash
pip install -q -U google-genai
```

```python
from google import genai

client = genai.Client()

response = client.models.generate_content(
    model="gemini-flash-latest", contents="Why do some birds migrate?"
)
```

### Step 2

In this step, your goal is send multiple requests to the Gemini API and time your code.

Run `generate_content()` in a loop with 5 iterations. Time your script. How long does it take to make 5 requests?

```bash
time python script.py
```

### Step 3

In this step, your goal is to use `asyncio` to send these requests concurrently.

Make sure you switch to the async implementation of the Gemini API:

```python
response = await client.aio.models.generate_content(
    model="gemini-flash-latest", contents="Why do some birds migrate?"
)
```

Now run and time your code. How long does it take?

```admonish hint title="Hint" collapsible=true
You need to use `asyncio.gather()`. Take a look at the basics section.
```

### Step 4

What happens if you increase the number of iterations? At what point do you hit rate limits?

### Going Further

* See the [Rate Limiters](rate-limiters.md) chapter to build your own rate limiter and avoid hitting rate limits.
* Creating `asyncio` tasks does *not* scale infinitely. If you create tens of thousands of tasks you can run out of memory or see worse performance due to scheduling overhead. If you have a large number of requests, it's better to use a producer-consumer queue and a fixed number of consumer tasks. See the [Data Pipelines](data-pipelines.md) to learn how to do this.

```admonish success title=""
**Now take some time to attempt the challenge before looking at the solution!**
```

---

## Solution

Below is a walkthrough of one possible solution. Your implementation may differ, and that's okay! The key concepts are using `async/await` with the Gemini API and `asyncio.gather()` for concurrency

### Step 1 - Solution

We have our API key.

### Step 2 - Solution

Let's call `generate_content()` in a loop:

```python
from google import genai
from google.genai.types import GenerateContentResponse

_NUM_REQUESTS = 5


def generate_content(index: int, client: genai.Client) -> GenerateContentResponse:
    response = client.models.generate_content(
        model="gemini-flash-latest", contents="Why do some birds migrate?"
    )
    print(f"Request {index} completed")
    return response


def main():
    client = genai.Client()
    results = [generate_content(i, client) for i in range(_NUM_REQUESTS)]


if __name__ == "__main__":
    main()
```

This takes about 30 seconds to run, including the time to connect to the client.

```bash
time python script.py
> Request 0 completed
> Request 1 completed
> Request 2 completed
> Request 3 completed
> Request 4 completed
> 
> real    0m32.134s
> user    0m1.733s
> sys     0m0.204s
```

### Step 3 - Solution

Now let's define some coroutines and use `asyncio.gather()` to create and schedule the tasks concurrently:

```python
import asyncio
from google import genai
from google.genai.types import GenerateContentResponse


_NUM_REQUESTS = 5


async def generate_content(index: int, client: genai.Client) -> GenerateContentResponse:
    response = await client.aio.models.generate_content(
        model="gemini-flash-latest", contents="Why do some birds migrate?"
    )
    print(f"Request {index} completed")
    return response


async def main():
    client = genai.Client()
    coroutines = [generate_content(i, client) for i in range(_NUM_REQUESTS)]
    results = await asyncio.gather(*coroutines)


if __name__ == "__main__":
    asyncio.run(main())
```

We only create the client once as we only need to establish one client connection.

This takes ~10 seconds to run, about three times faster than before. The requests run concurrently and don't return in the same order.

```bash
time python script.py
> Request 3 completed
> Request 4 completed
> Request 1 completed
> Request 0 completed
> Request 2 completed
> 
> real    0m8.460s
> user    0m1.860s
> sys     0m0.240s
```

## Step 4 - Solution

If we increase `_NUM_REQUESTS = 20` we quickly hit the rate limit.

```bash
google.genai.errors.ClientError: 429 RESOURCE_EXHAUSTED. {'error': {'code': 429, 
 'message': 'You exceeded your current quota, please check your plan and billing details....
```
