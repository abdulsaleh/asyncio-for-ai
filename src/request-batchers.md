# Request Batchers

## Challenge

In this challenge, you will build an async request batcher for the [Gemini Embedding API](https://ai.google.dev/gemini-api/docs/embeddings).

Many applications require batching API calls for efficiency. Say you're building an application that requires generating embeddings for a stream of queries (e.g., a search engine).

You have two options:

1. **Generate embeddings one at a time** - This is simple but each request to the embedding generation service incurs some network latency and overhead.
2. **Batch multiple requests together** - Combine several inputs into a single request and minimize the network overhead.

In this challenge, you will batch requests based on two parameters:

- **`batch_size`** - Maximum number of requests in a batch
- **`timeout`** - Maximum time to wait to fill a batch

The batcher continuously reads from an input queue and creates a batch when the batch size is reached or when the timeout expires (resulting in a partial batch).

```admonish info
In real systems, the batch size and timeout are tuned to balance efficiency and latency. Large batches are generally more efficient but waiting to fill them can hurt end-to-end latency.
```

We will be using producer-consumer queues in the challenge. See the [Data Pipelines](data-pipelines.md) chapter to learn more about this pattern.

In this challenge, you will implement four async functions or coroutines:

1. An input enqueuer that adds inputs to the input queue.
2. A batcher that reads from the input queue, creates batches, then adds them to the batch queue.
3. An embedding generator that reads from the batch queue, calls the Gemini Embedding API, then adds the output to the output queue.
4. An output logger that reads from the outputs queue and prints the outputs.

## Before you start

The following functions or classes are relevant for this chapter. It might be helpful to read their docs before you start:

* `asyncio.Queue()` for creating async queues.
* `asyncio.wait_for()` for implementing timeouts on async operations.
* `asyncio.get_running_loop().time()` for tracking elapsed time.
* `queue.task_done()` and `queue.join()` for tracking queue completion.
* `task.cancel()` for canceling running tasks.

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

### Step 2

In this step, your goal is to implement the input enqueuer coroutine.

This coroutine simulates a stream of inputs arriving at variable times. It adds inputs to the input queue with random delays between each input.

```python
import random

_NUM_INPUTS = 100

async def enqueue_inputs(input_queue: asyncio.Queue[str]):
    for i in range(_NUM_INPUTS):
        await input_queue.put(f"input-{i}")
        # Simulate variable input arrival times
        await asyncio.sleep(random.uniform(0, 0.3))
```

This simulates real-world scenarios where inputs don't arrive all at once but stream in over time.

### Step 3

In this step, your goal is to implement the `Batcher` class.

The batcher is the core of this challenge. It reads from the input queue and creates batches based on two conditions:

1. **Batch size reached** - When we have collected `batch_size` inputs, send the batch immediately.
2. **Timeout expired** - If the timeout expires before the batch fills, send a partial batch.

```python
from datetime import timedelta

class Batcher:
    def __init__(self, batch_size: int, timeout: timedelta):
        self._batch_size = batch_size
        self._timeout = timeout.total_seconds()

    async def batch(
        self, input_queue: asyncio.Queue[str], batch_queue: asyncio.Queue[list[str]]
    ):
        pass
```

The `batch()` method should:

1. Use a `while True` loop to continuously process inputs.
2. Track the remaining timeout for the current batch. Use `asyncio.get_running_loop().time()` to get the current time.
3. Use `asyncio.wait_for(input_queue.get(), timeout)` to wait for an input with a timeout. This will raise `asyncio.TimeoutError` if the timeout expires.
4. If the batch size is reached or the timeout expires, add the batch to the batch queue.
5. Handle `asyncio.CancelledError` to shut down gracefully.

```admonish warning title="Important"
Make sure to update the remaining timeout after each input. As time elapses, the remaining timeout decreases. This ensures the batch times out at the correct time.
```

### Step 4

In this step, your goal is to implement the embedding generator coroutine.

This coroutine reads batches from the batch queue, calls the Gemini Embedding API, then adds individual (content, embedding) pairs to the output queue.

```python
from google.genai import types

async def embed_content(
    client: genai.Client,
    batch_queue: asyncio.Queue[list[str]],
    output_queue: asyncio.Queue[tuple[str, types.ContentEmbedding]],
):
    pass
```

The Gemini API accepts multiple contents in a single call and returns embeddings in the same order. You need to zip the batch contents with the returned embeddings then add them to the output queue individually.

### Step 5

In this step, your goal is to implement the output logger coroutine.

This coroutine reads from the output queue and prints each (content, embedding) pair. We could update this to write the results to a file, but we will just print the output here for simplicity.

```python
async def log_outputs(
    output_queue: asyncio.Queue[tuple[str, types.ContentEmbedding]],
):
    pass
```

### Step 6

In this step, your goal is to chain all the coroutines together in the main function.

You need to create three queues:

```python
async def main():
    input_queue = asyncio.Queue()
    batch_queue = asyncio.Queue()
    output_queue = asyncio.Queue()
```

Then create the tasks:

1. Create the input enqueuer task.
2. Create the batcher task with a batch size of 8 and timeout of 1 second.
3. Create the embedding generator task.
4. Create the output logger task.

After creating the tasks, you need to wait for them to complete in the correct order and gracefully handle shutdown.

Run your code and verify that:

- Batches are created when they reach size 8.
- Partial batches are created when the timeout expires.
- All inputs are processed even if the final batch is partial.

## Going Further

- Try experimenting with different batch sizes and timeouts. How does this affect the total runtime? What's the tradeoff between efficiency and latency?

- Implement a dynamic batching strategy that adjusts the batch size based on input arrival rate. If inputs are arriving slowly, use smaller batches with shorter timeouts. If inputs are arriving quickly, use larger batches.

```admonish success title=""
**Now take some time to attempt the challenge before looking at the solution!**
```

---

## Solution

Below is a walkthrough of one possible solution. Your implementation may differ, and that's okay!

First let's define all the imports and type aliases:

```python
import asyncio
import random
from datetime import timedelta
from typing import TypeAlias

from google import genai
from google.genai import types

_NUM_INPUTS = 100

Input = str
InputBatch: TypeAlias = list[Input]
```

### Step 2 - Solution

```python
async def enqueue_inputs(input_queue: asyncio.Queue[Input]):
    for i in range(_NUM_INPUTS):
        await input_queue.put(f"input-{i}")
        # Simulate variable input arrival times
        await asyncio.sleep(random.uniform(0, 0.3))
    print("All inputs enqueued.")
```

This coroutine adds inputs to the queue with random delays. The random delays simulate real-world scenarios where inputs arrive at variable rates.

### Step 3 - Solution

```python
class Batcher:
    def __init__(self, batch_size: int, timeout: timedelta):
        self._batch_size = batch_size
        self._timeout = timeout.total_seconds()

    async def batch(
        self, input_queue: asyncio.Queue[Input], batch_queue: asyncio.Queue[InputBatch]
    ):
        try:
            loop = asyncio.get_running_loop()
            while True:
                batch = []
                start = loop.time()
                timeout = self._timeout
                while True:
                    try:
                        item = await asyncio.wait_for(input_queue.get(), timeout)
                    except asyncio.TimeoutError:
                        # Batch timed out.
                        elapsed = loop.time() - start
                        print(f"Batch timed out after {elapsed} seconds.")
                        break

                    batch.append(item)

                    # Update the timeout since some time has elapsed.
                    elapsed = loop.time() - start
                    timeout = self._timeout - elapsed

                    if len(batch) == self._batch_size:
                        # Batch is full.
                        print('Batch is full.')
                        break

                if batch:
                    print(f"Batch size is {len(batch)}")
                    await batch_queue.put(batch)
                    for _ in batch:
                        input_queue.task_done()

        except asyncio.CancelledError:
            print("Shutting down batcher...")
```

Note how:

- We track the elapsed time and update the remaining timeout after each input. This ensures batches time out at the correct time regardless of when inputs arrive.
- We use `asyncio.wait_for()` to implement the timeout. This raises `asyncio.TimeoutError` when the timeout expires.
- We call `input_queue.task_done()` for each item in the batch to properly track queue completion.
- Empty batches are skipped to avoid sending empty API requests.

### Step 4 - Solution

```python
async def embed_content(
    client: genai.Client,
    batch_queue: asyncio.Queue[InputBatch],
    output_queue: asyncio.Queue[tuple[Input, types.ContentEmbedding]],
):
    try:
        while True:
            batch = await batch_queue.get()
            result = await client.aio.models.embed_content(
                model="gemini-embedding-001",
                contents=batch,
                config=types.EmbedContentConfig(output_dimensionality=3),
            )
            if result.embeddings is None:
                raise ValueError("No embeddings returned from the API.")

            for content, embedding in zip(batch, result.embeddings):
                await output_queue.put((content, embedding))

            batch_queue.task_done()

    except asyncio.CancelledError:
        print("Shutting down embedding task...")
```

The Gemini API returns embeddings in the same order as the input contents. We use `zip()` to pair each content with its embedding, then add them individually to the output queue.

### Step 5 - Solution

```python
async def log_outputs(
    output_queue: asyncio.Queue[tuple[Input, types.ContentEmbedding]],
):
    try:
        while True:
            content, embedding = await output_queue.get()
            print(f"Content: {content} -> Embedding: {embedding.values}")
            output_queue.task_done()
    except asyncio.CancelledError:
        print("Shutting down writer...")
```

### Step 6 - Solution

```python
async def main():
    input_queue = asyncio.Queue()
    batch_queue = asyncio.Queue()
    output_queue = asyncio.Queue()

    client = genai.Client()
    batcher = Batcher(batch_size=8, timeout=timedelta(seconds=1))

    enqueue_task = asyncio.create_task(enqueue_inputs(input_queue))
    batcher_task = asyncio.create_task(batcher.batch(input_queue, batch_queue))
    embed_task = asyncio.create_task(embed_content(client, batch_queue, output_queue))
    log_task = asyncio.create_task(log_outputs(output_queue))

    # Wait until the elements in each queue have been processed, then cancel tasks.
    await enqueue_task
    await input_queue.join()

    batcher_task.cancel()
    await batcher_task

    await batch_queue.join()

    embed_task.cancel()
    await embed_task

    await output_queue.join()

    log_task.cancel()
    await log_task


asyncio.run(main())
```

This is how the shutdown sequence works:

1. First, wait for all inputs to be enqueued (`await enqueue_task`).
2. Wait for the input queue to be empty (`await input_queue.join()`), then cancel the batcher since there are no more inputs.
3. Wait for the batch queue to be empty (`await batch_queue.join()`), then cancel the embedding task since there are no more batches.
4. Wait for the output queue to be empty (`await output_queue.join()`), then cancel the logger since there are no more outputs.

This ensures graceful shutdown without losing any data.

```admonish info
Another option is to pass a "poison pill" or a None sentinel to notify the consumers to shut down instead of cancelling them explicitly. 
```

Now let's run this and check the output:

```bash
...
Batch timed out after 1.0009840049997365 seconds.
Batch size is 5
Content: input-49 -> Embedding: [-0.011579741, -0.010578066, 0.018000955]
Content: input-50 -> Embedding: [-0.004875405, -3.5398414e-05, 0.019322932]
Content: input-51 -> Embedding: [-0.014710601, -0.005792096, 0.0061859917]
Content: input-52 -> Embedding: [-0.01397081, -0.0063333134, 0.01519548]
Content: input-53 -> Embedding: [0.00022102315, -0.010211905, 0.012408208]
Batch is full.
Batch size is 8
Content: input-54 -> Embedding: [-0.005042672, -0.010983077, 0.011013798]
Content: input-55 -> Embedding: [-0.013474228, -0.0002503009, 0.021384422]
Content: input-56 -> Embedding: [-0.007262096, -0.012684701, 0.02125588]
Content: input-57 -> Embedding: [-0.007679552, -0.0016124722, 0.014986247]
Content: input-58 -> Embedding: [-0.0013965481, -0.016043654, 0.015830107]
Content: input-59 -> Embedding: [0.0011211497, 0.0062395, 0.022803845]
Content: input-60 -> Embedding: [-0.005757582, -0.019661691, 0.012294045]
Content: input-61 -> Embedding: [-0.009954969, -0.030515866, 0.0075060125]
....
Batch timed out after 1.0012599780002347 seconds.
Batch size is 4
Content: input-90 -> Embedding: [-0.0027523814, -0.015859226, 0.017288134]
Content: input-91 -> Embedding: [-0.0006344775, -0.019411083, 0.00752806]
Content: input-92 -> Embedding: [-0.010482627, -0.002565886, 0.028898504]
Content: input-93 -> Embedding: [0.0059372373, -0.015417972, 0.016691986]
All inputs enqueued.
Batch timed out after 1.0011447710003267 seconds.
Batch size is 6
Shutting down batcher...
Content: input-94 -> Embedding: [-0.010927118, -0.014871426, 0.01482102]
Content: input-95 -> Embedding: [-0.0030360888, -0.007296145, 0.009159293]
Content: input-96 -> Embedding: [-0.01636001, -0.0077388645, 0.015982967]
Content: input-97 -> Embedding: [-0.0011909363, -0.01127491, 0.013533601]
Content: input-98 -> Embedding: [-0.005317036, -0.024050364, 0.010049778]
Content: input-99 -> Embedding: [-0.010439553, 0.0002079097, 0.027198879]
Shutting down embedding task...
Shutting down writer...
```

Note how:

- Some batches reach size 8 and are sent immediately (marked as "Batch is full").
- Other batches timeout after 1 second and are sent as partial batches.
- Inputs, batching, API calls, and logging are all interleaved and happening concurrently.
- The batcher automatically adapts to the input arrival rate, creating full batches when inputs arrive quickly and partial batches when they arrive slowly.
