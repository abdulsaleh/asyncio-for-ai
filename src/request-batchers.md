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

This coroutine simulates a stream of inputs arriving at variable times. It adds inputs to the input queue with random delays between each input. After all inputs are added, it sends a **sentinel value** (`None`) to signal the end of the input stream.

```python
import random

_NUM_INPUTS = 100

async def enqueue_inputs(input_queue: asyncio.Queue[str | None]):
    for i in range(_NUM_INPUTS):
        await input_queue.put(f"input-{i}")
        # Simulate variable input arrival times
        await asyncio.sleep(random.uniform(0, 0.3))
    # Signal end of inputs with sentinel value
    await input_queue.put(None)
```

This simulates real-world scenarios where inputs don't arrive all at once but stream in over time. The sentinel value is a clean way to signal completion without needing to cancel tasks.

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
        self, input_queue: asyncio.Queue[str | None], batch_queue: asyncio.Queue[list[str] | None]
    ):
        pass
```

The `batch()` method should:

1. Use a `while True` loop to continuously process inputs.
2. Track the remaining timeout for the current batch. Use `asyncio.get_running_loop().time()` to get the current time.
3. Use `asyncio.wait_for(input_queue.get(), timeout)` to wait for an input with a timeout. This will raise `asyncio.TimeoutError` if the timeout expires.
4. If the batch size is reached or the timeout expires, add the batch to the batch queue.
5. Check for the sentinel value (`None`). When received, send any remaining batch, then propagate the sentinel to the next queue and return.

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
    batch_queue: asyncio.Queue[list[str] | None],
    output_queue: asyncio.Queue[tuple[str, types.ContentEmbedding] | None],
):
    pass
```

The Gemini API accepts multiple contents in a single call and returns embeddings in the same order. You need to zip the batch contents with the returned embeddings then add them to the output queue individually. When you receive the sentinel value, propagate it to the output queue and return.

### Step 5

In this step, your goal is to implement the output logger coroutine.

This coroutine reads from the output queue and prints each (content, embedding) pair. We could update this to write the results to a file, but we will just print the output here for simplicity.

```python
async def log_outputs(
    output_queue: asyncio.Queue[tuple[str, types.ContentEmbedding] | None],
):
    pass
```

Check for the sentinel value and return when received.

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

After creating the tasks, you can wait for all of them to complete using `asyncio.gather()`. With sentinel values, tasks will shut down gracefully on their own without cancellation.

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
from typing import AsyncIterator, TypeAlias

from google import genai
from google.genai import types

_NUM_INPUTS = 100
_BATCH_SIZE = 8
_TIMEOUT_SECONDS = 1
_INPUT_DELAY_SECONDS = 0.3


Input: TypeAlias = str
InputBatch: TypeAlias = list[Input]
```

### Step 2 - Solution

```python
async def enqueue_inputs(input_queue: asyncio.Queue[Input | None]):
    for i in range(_NUM_INPUTS):
        await input_queue.put(f"input-{i}")
        # Simulate variable input arrival times
        await asyncio.sleep(random.uniform(0, _INPUT_DELAY_SECONDS))
    # Signal end of inputs with sentinel value
    await input_queue.put(None)
    print("All inputs enqueued.")
```

This coroutine adds inputs to the queue with random delays. The random delays simulate real-world scenarios where inputs arrive at variable rates. After all inputs are enqueued, we send a sentinel value (`None`) to signal that no more inputs are coming. This allows downstream consumers to shut down gracefully without explicit cancellation.

### Step 3 - Solution

```python
class Batcher:
    def __init__(self, batch_size: int, timeout: timedelta):
        self._batch_size = batch_size
        self._timeout = timeout.total_seconds()

    async def _batches(
        self, input_queue: asyncio.Queue[Input | None]
    ) -> AsyncIterator[InputBatch | None]:
        """Yields batches until sentinel received. Yields None as final value."""
        loop = asyncio.get_running_loop()

        while True:
            batch = []
            start = loop.time()
            timeout = self._timeout

            while True:
                try:
                    item = await asyncio.wait_for(input_queue.get(), timeout)
                except asyncio.TimeoutError:
                    elapsed = loop.time() - start
                    print(f"Batch timed out after {elapsed} seconds.")
                    break

                if item is None:
                    # Received sentinel value.
                    input_queue.task_done()
                    if batch:
                        yield batch
                    yield None
                    return

                batch.append(item)
                elapsed = loop.time() - start
                timeout = self._timeout - elapsed

                if len(batch) == self._batch_size:
                    print("Batch is full.")
                    break

            if batch:
                yield batch
                for _ in batch:
                    input_queue.task_done()

    async def batch(
        self,
        input_queue: asyncio.Queue[Input | None],
        batch_queue: asyncio.Queue[InputBatch | None],
    ):
        async for batch in self._batches(input_queue):
            if batch is None:
                # Propagate sentinel to next stage
                await batch_queue.put(None)
                print("Shutting down batcher...")
                return
            print(f"Batch size is {len(batch)}")
            await batch_queue.put(batch)
```

Some things to note:

- We track the elapsed time and update the remaining timeout after each input. This ensures batches time out at the correct time regardless of when inputs arrive.
- We use `asyncio.wait_for()` to implement the timeout. This raises `asyncio.TimeoutError` when the timeout expires.
- We check for the sentinel value (`None`). When received, we send any remaining batch, propagate the sentinel to the next queue, and return. This gracefully shuts down the batcher without needing task cancellation.
- We call `input_queue.task_done()` for each item in the batch to properly track queue completion.
- Empty batches are skipped to avoid sending empty API requests.

### Step 4 - Solution

```python
async def embed_content(
    client: genai.Client,
    batch_queue: asyncio.Queue[InputBatch | None],
    output_queue: asyncio.Queue[tuple[Input, types.ContentEmbedding] | None],
):
    while True:
        batch = await batch_queue.get()

        if batch is None:
            # Received sentinel value.
            break

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

    # Propagate sentinel to next stage
    await output_queue.put(None)
    batch_queue.task_done()
    print("Shutting down embedding task...")
```

The Gemini API returns embeddings in the same order as the input contents. We use `zip()` to pair each content with its embedding, then add them individually to the output queue. When we receive the sentinel value, we propagate it to the output queue and return.

### Step 5 - Solution

```python
async def log_outputs(
    output_queue: asyncio.Queue[tuple[Input, types.ContentEmbedding] | None],
):
    while True:
        output = await output_queue.get()

        if output is None:
            # Received sentinel value.
            break

        content, embedding = output
        print(f"Content: {content} -> Embedding: {embedding.values}")
        output_queue.task_done()

    output_queue.task_done()
    print("Shutting down writer...")
```

### Step 6 - Solution

```python
async def main():
    input_queue = asyncio.Queue()
    batch_queue = asyncio.Queue()
    output_queue = asyncio.Queue()

    client = genai.Client()
    batcher = Batcher(
        batch_size=_BATCH_SIZE, timeout=timedelta(seconds=_TIMEOUT_SECONDS)
    )

    enqueue_task = asyncio.create_task(enqueue_inputs(input_queue))
    batcher_task = asyncio.create_task(batcher.batch(input_queue, batch_queue))
    embed_task = asyncio.create_task(embed_content(client, batch_queue, output_queue))
    log_task = asyncio.create_task(log_outputs(output_queue))

    # Wait for all tasks to complete gracefully via sentinel values
    await asyncio.gather(enqueue_task, batcher_task, embed_task, log_task)


asyncio.run(main())
```

This is how the shutdown sequence works using sentinel values:

1. The input enqueuer finishes adding all inputs, then sends a sentinel value (`None`) to the input queue.
2. The batcher receives the sentinel, sends any remaining batch, propagates the sentinel to the batch queue, and returns.
3. The embedding task receives the sentinel, propagates it to the output queue, and returns.
4. The logger receives the sentinel and returns.
5. All tasks complete naturally, and `asyncio.gather()` returns.

This approach is cleaner than explicit cancellation because each stage knows when to shut down based on the sentinel value.

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
