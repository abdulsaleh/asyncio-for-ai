# Async LLM Responses


## Challenge

This challenge is to build an asynchronous LLM client. 



### Step 0

Get a Gemini [API key](https://aistudio.google.com/app/api-keys) from Google AI Studio. We use the Gemini API because it has a generous free tier. 


### Step 1
In this step, your goal is to verify you can call the Gemini API and get some responses. 

Install the Google [GenAI SDK](https://ai.google.dev/gemini-api/docs/quickstart) and make your first request. Write your code to `script.py`.

```bash
pip install -q -U google-genai
```


```python
from google import genai

client = genai.Client(api_key="YOUR_API_KEY")

response = client.models.generate_content(
    model="gemini-flash-latest", contents="Why do some birds migrate?"
)
```

### Step 2

In this step, your goal is send multiple requests to the Gemini API and time your code.

Run `generate_content()` in a loop with 10 iterations. Time your script. How long does it take to make 10 requests?

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

<details>
    <summary>Hint</summary>
    You need to use `asyncio.gather()`. Take a look at the basics section.
</details>


### Step 4

What happens if you increase the number of iterations? At what point do you hit rate limits?

See the [Rate Limiters](#rate-limiters.md) chapter to build your own rate limiter and avoid hitting rate limits.


## Solution

### Step 1
We have our API key. 

### Step 2

Let's call `generate_content()` in a loop:

```python
import asyncio
from google import genai

_NUM_REQUESTS = 10

def generate_content(client):
    response = await client.models.generate_content(
    model="gemini-flash-latest", contents="Why do some birds migrate?"
)

def main():
    client = genai.Client()
    results = [generate_content(client) for _ in range(_NUM_REQUESTS)]
    


if __name__ == "__main__":
    main()
```

This takes about seconds to run.

```bash
time python script.py
> 
```

### Step 3

Now let's define some coroutines and use `asyncio.gather()` to create and schedule the tasks concurrently:

```python
import asyncio 

_NUM_REQUESTS = 10

async def generate_content(client):
    response = await client.aio.models.generate_content(
    model="gemini-flash-latest", contents="Why do some birds migrate?"
)

async def main():
    client = genai.Client()
    coroutines = [generate_content(client) for _ in range(_NUM_REQUESTS)]
    results = await asyncio.gather(*coroutines)


if __name__ == "__main__":
    asyncio.run(main())
```

We only create the client once. We only need to establish the client connection once. Creating multiple clients would add uneccessary overhead. 


```bash
time python script.py
> 
```

## Step 4

If we increase `_NUM_REQUESTS` we quickly hit the rate limit.

```

```