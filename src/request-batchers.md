# Request Batchers

## Challenge

In this challenge, you will build an asynchronous request batcher for the [Gemini Embedding API](https://ai.google.dev/gemini-api/docs/embeddings).

Many applications require batching API calls. For example, say you're building a search engine with [vector retrieval](https://www.oracle.com/database/vector-search/). You need to generate an embedding for each request, and then use that embedding to find relevant search results in a vector database. This setup is also very common for LLM [RAG](https://writer.com/engineering/rag-vector-database/) pipelines.

You'll be receiving a stream of requests. You can generate these embeddings one at a time by calling an external service like the Gemini Embedding API. The problem is that each request has some overhead from network latency. Sending many small requests can also push you over the rate limit.

A batcher allows you to combine multiple requests into one to minimize this overhead.

The batcher in this challenge has two parameters: `batch_size` and batch `timeout`. As requests arrive in a stream, the batcher populates the batch until either hit the batch size or hit the timeout (in which case we have a partially filled batch).

In real applications, the batch size and timeout are tuned to trade off efficiency for latency. Large batches are more efficient but waiting to fill those batches can hurt end-to-end latency.

## Producer-Consumer Queues

You will be using a producer-consumer queue in this challenge, which is a key design pattern when writing concurrent code.

**A queue allows different tasks to communicate with each other.** A producer task can add items to a queue, and a consumer task can process these items as they are added. 

Take a look at this example:

```python
import asyncio

_NUM_ITEMS = 3


async def producer(queue: asyncio.Queue):
    for i in range(_NUM_ITEMS):
        print(f"Adding item {i}")
        await queue.put(i)
        # Pass control back to the event loop so the consumer can run.
        await asyncio.sleep(0.1)


async def consumer(queue: asyncio.Queue):
    try:
        while True:
            item = await queue.get()
            print(f"Processing the item {item}")
            queue.task_done()

    except asyncio.CancelledError:
        print("Shutting down consumer.")


async def main():
    queue = asyncio.Queue()

    consumer_task = asyncio.create_task(consumer(queue))
    producer_task = asyncio.create_task(producer(queue))

    # Wait until all the producers are done.
    # This ensures all items have been added to the queue.
    await producer_task

    # Wait until the queue is empty.
    # .put() and .task_done() must be called an equal number of times.
    await queue.join()

    # Cancel the consumers since the queue is empty.
    consumer_task.cancel()
    await consumer_task


if __name__ == "__main__":
    asyncio.run(main())
```

The code above produces the following output:

```bash
> Adding item 0
> Processing the item 0
> Adding item 1
> Processing the item 1
> Adding item 2
> Processing the item 2
> Shutting down consumer.
```

Note how the producer and consumer steps are **interleaved**. Items are processed as they are added. This is the beauty of concurrent code. 


```admonish warning title="Important"
The consumer loops forever waiting on new items. You need to cancel the consumer tasks explicitly after the producers are done and the queue is empty.
```


You can create multiple consumer or producer tasks all using the same queue. This is analogous to creating more "workers" to achieve higher concurrency. 

```python
_NUM_PRODUCERS = 4
_NUM_CONSUMERS = 2

async def main():
    queue = asyncio.Queue()

    consumers = [asyncio.create_task(consumer(queue)) for _ in range(_NUM_CONSUMERS)]
    producers = [asyncio.create_task(producer(queue)) for _ in range(_NUM_PRODUCERS)]

    await asyncio.gather(*producers)
    await queue.join()

    for c in consumers:
        c.cancel()
    await asyncio.gather(*consumers)
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
