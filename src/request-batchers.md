# Request Batchers

## Challenge

In this challenge, you will build an async request batcher for the [Gemini Embedding API](https://ai.google.dev/gemini-api/docs/embeddings).

Many applications require batching API calls for efficiency. Say you're building an application such as a search engine that requires generating embeddings for a stream of queries.

You have two options:

1. **Generate embeddings one at a time** - This is simple but each API call or request to the embedding generation service has some network latency and overhead.
2. **Batch multiple requests together** - Combine several inputs into a single API call and minimize the network overhead.

In this challenge, you will batch requests based on two parameters:

- **`batch_size`** - Maximum number of requests in a batch
- **`timeout`** - Maximum time to wait for a batch to fill

The batcher continuously reads from an input queue and creates a batch when the batch size is reached or when the timeout expires (resulting in a partial batch).

```admonish info
In real systems, the batch size and timeout are tuned to balance efficiency and latency. Large batches are generally more efficient but waiting to fill them can hurt end-to-end latency.
```

We will be using producer-consumer queues in the challenge. See the [Data Pipelines](data-pipelines.md) chapter to learn more about this pattern.

In this challenge, you will implement four async functions or coroutines:

1. An input reader that adds inputs to the input queue.
2. A batcher that reads from the input queue, creates batches, then adds them to the batch queue.
3. An embedding generator that reads from the batch queue, calls the Gemini Embedding API, then adds the output to the output queue.
4. An output logger that reads from the outputs queue and logs the outputs.

### Step 0

To get started, get a Gemini API key from [Google AI Studio](https://aistudio.google.com/app/api-keys). We use the Gemini API because it has a generous free tier.

```bash
export GEMINI_API_KEY="YOUR_API_KEY"
```

### Step 1

In this step, your goal is to verify you can call the Gemini API to generate embeddings.

Install the Google [GenAI SDK](https://ai.google.dev/gemini-api/docs/quickstart) and make your first request. Write your code to `script.py`.

```bash
pip install -q -U google-genai
```

```python
import asyncio

from google import genai

_NUM_REQUESTS = 10


async def main():
    client = genai.Client()
    contents = [f"Input: {i}" for i in range(_NUM_REQUESTS)]
    result = await client.aio.models.embed_content(
        model="gemini-embedding-001", contents=contents
    )

if __name__ == "__main__":
    asyncio.run(main())
```
