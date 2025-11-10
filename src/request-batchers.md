# Request Batchers

## Challenge

In this challenge, you will build an asynchronous request batcher for the [Gemini Embedding API](https://ai.google.dev/gemini-api/docs/embeddings).

Many applications require batching API calls. For example, let's say you're building a search engine with vector retrieval. You need to generate an embedding for each request, and then use that embedding to find relevant search results. This is common if you're doing RAG with a vector DB for example.

You'll be receiving a stream of requests. You can generate these embeddings one at a time by calling an external service like the Gemini Embedding API. The problem is that each request has some overhead from network latency. Sending many small requests can also push you over the rate limit.

A batcher allows you to combine multiple requests into one to minimize this overhead.

The batcher in this challenge has two parameters: `batch_size` and batch `timeout`. As requests arrive in a stream, the batcher populates the batch until either hit the batch size or hit the timeout (in which case we have a partially filled batch).

In real applications, the batch size and timeout are tuned to trade off efficiency for latency. Large batches are more efficient but waiting to fill those batches can hurt end-to-end latency.

## Producer-Consumer Queues

Explain how queues work.

```python
import asyncio


_NUM_ITEMS = 10


async def producer(queue: asyncio.Queue):
    for i in range(_NUM_ITEMS):
        # Pass control back to the event loop.
        # The consumer can then run concurrently.
        await queue.put(i)


async def consumer(queue: asyncio.Queue):
    try:
        while True:
            item = await queue.get()
            print('Processing the item...')
            
            queue.task_done()
            print('Done processing the item.')
    
    except asyncio.CancelledError:
        print('Shutting down consumer.')


async def main():
    queue = queue.Queue()

    consumer_task = asyncio.create_task(consumer(queue))
    producer_task = asyncio.create_task(producer(queue))

    # Wait until all the producers are done.
    # This ensures all items have been added to the queue.
    await producer_task

    # Wait until the queue is empty.
    # Wait until .put() and .task_done() are called an equal number of times.
    await queue.join()

    # We then cancel the consumers since the queue is empty.
    consumer_task.cancel()
    await consumer_task


asyncio.run(main())
```

### Step 0

To get started, get a Gemini API key from [Google AI Studio](https://aistudio.google.com/app/api-keys). We use the Gemini API because it has a generous free tier.

```bash
export GEMINI_API_KEY="YOUR_API_KEY"
```

### Step 1

In this step, your goal is to verify you can call the API and generate embeddings.

Install the Google [GenAI SDK](https://ai.google.dev/gemini-api/docs/quickstart) and make your first request. Write your code to `script.py`.

```bash
pip install -q -U google-genai
```

```python
from google import genai

client = genai.Client()

result = client.models.embed_content(
        model="gemini-embedding-001",
        contents="What is the meaning of life?")
```

### Step 2

In this step, your goal is to implement the asynchronous request batcher.

### Step 3

In this step, your goal is to implement the embedding worker.

This worker calls the Gemini Embedding API to generate embeddings.

### Step 4

In this step, your goal is to build a producer-consumer pipeline

## Solution
