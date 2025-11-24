# Data Pipelines

## Challenge

In this challenge, you will build a concurrent data pipeline. The pipeline will read some prompts from disk, call the Gemini API, then write the model responses back to disk.

Reading/writing files and sending API requests are I/O operations so `asyncio` is a great fit for speeding up this pipeline. Instead of waiting on disk I/O or model responses, you can be doing work concurrently to save time.

You can also extend this pipeline to do CPU-bound work. See the [Going Further](#going-further) section if you want to learn how.

## Before you start

The following functions or classes are relevant for this chapter. It might be helpful to read their docs before you start:

* `asyncio.gather()` for waiting on running tasks.
* `asyncio.Queue()` for creating async queues.
* `task.cancel()` for canceling running tasks.
* `aiofiles.open()` for async file operations.

## Producer-Consumer Queues

You will be using a producer-consumer queue in this challenge, which is a key design pattern for concurrent code.

**A queue allows different tasks to communicate with each other.** A producer task can add items to a queue, and a consumer task can process these items as they are added.

Queues are useful when you have:

1. **Different pipeline stages** - API calls are slow, but file I/O is fast. We can keep reading/enqueuing files while the API calls are inflight.
2. **Multiple workers per stage** - Multiple workers can process API calls concurrently, finishing faster.
3. **Variable processing times** - Some API calls return in 100ms, others take 2s. Queues keep all workers busy.
4. **Backpressure control** - Limiting the queue sizes prevents memory overflow if producers are faster than consumers.

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

Note how the producer and consumer steps are **interleaved**. Items are processed as they are added.

The consumer doesn't wait for all items to be produced. It starts processing immediately. With multiple consumers, you can process many items concurrently while the producer is still adding more.

You can create **multiple** consumer or producer tasks all using the same queue. This is analogous to creating more "workers" to achieve higher concurrency.

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

```admonish warning title="Important"
The consumers loop forever waiting on new items. You need to cancel the consumers explicitly after the producers are done and the queue is empty.
```

---
---

The data pipeline in this challenge will have:

1. An **input reader** task which reads input prompts from disk then adds them to the input queue.
2. Multiple **content generation** tasks which get prompts from the input queue, call the Gemini API, then add responses to the output queue.
3. An **output writer** task which gets the responses from the outputs queue then writes them to disk.

The queue decouples stages so they don't block each other. The content generation workers also process API calls concurrently so we can have multiple API calls inflight at the same time.

### Step 0

To get started, get a Gemini API key from [Google AI Studio](https://aistudio.google.com/app/api-keys). We use the Gemini API because it has a generous free tier, but any async model API will work.

```bash
export GEMINI_API_KEY="YOUR_API_KEY"
```

### Step 1

In this step, your goal is to generate the input prompts.

Run the command below to create a new `inputs/` directory. This directory will have 10 files, each with 3 prompts. This will simulate reading from sharded input files.

```bash
mkdir -p inputs

for shard in {0..9}; do for i in {0..2}; do echo "What is two times $((shard * 3 + i))?" >> inputs/shard_$shard.txt;
done; done
```

Verify that the input data was created:

```bash
ls -l inputs

>-rw-rw-r-- 1 user user 63 Nov 23 16:34 shard_0.txt
>-rw-rw-r-- 1 user user 63 Nov 23 16:34 shard_1.txt
>-rw-rw-r-- 1 user user 63 Nov 23 16:34 shard_2.txt
...
```

Also inspect the input files:

```bash
cat inputs/shard_4.txt 

> What is two times 12?
> What is two times 13?
> What is two times 14?
```

### Step 2

In this step, your goal is to implement the input reader coroutine.

The input reader coroutine takes the input directory, reads the files in that directory, then adds the file contents to the input queue. Each line in the file is a separate prompt or input.

You can use `aiofiles` for async file operations. The input directory in our case is `inputs/`.

```python
import aiofiles

async def read_inputs(input_dir: pathlib.Path, input_queue: asyncio.Queue):
    pass
```

### Step 3

In this step, your goal is to implement the content generation coroutine.

This coroutine reads prompts from the input queue, makes requests to the Gemini API, then writes the response text to the output queue.

See the solution to the [LLM Responses](llm-responses.md) chapter to see how to call the async Gemini API.

```python
async def generate_content(
    client: genai.Client, input_queue: asyncio.Queue, output_queue: asyncio.Queue
):
    pass
```

Make sure to handle the API call failures. You can either skip failed inputs, retry them in the current task, or add them back to the input queue to be processed by another task.

```admonish tip title="Tip"
To avoid hitting rate limits, you can sleep between requests or you can use the rate limiter we implemented in the [Rate Limiters](#rate-limiters.md) chapter.
```

### Step 4

In this step, your goal is to implement the output writer coroutine.

This coroutine does the following:

1. It creates an output shard file at the output directory.
2. It gets responses from the output queue, then writes them to the output file until `_SHARD_SIZE` is reached.
3. After the shard is full, we create a new shard and repeat.

You can use `aiofiles` again for file operations.

```python
_SHARD_SIZE = 5

async def write_outputs(output_dir: pathlib.Path, output_queue: asyncio.Queue):
    return 
```

```admonish warning title="Important"
Make sure there is only one task writing to any given file at a time. Having multiple tasks write to the same file or shard can cause race conditions.
```

### Step 5

In this step, your goal is to create the final consumer-producer pipeline, chaining all the coroutines you have implemented above.

You need to define two queues:

```python
_QUEUE_SIZE = 5

async def main():
    inputs_queue = asyncio.Queue(maxsize=_QUEUE_SIZE)
    outputs_queue = asyncio.Queue(maxsize=_QUEUE_SIZE)
```

Set the `maxsize` parameter for these queues to implement backpressure. This prevents the reader from loading all prompts into memory at once and causing memory overflow. When a queue is full, `await queue.put()` will block until space is available.

You then need to create the tasks or workers:

1. Create one input reader task.
2. Create at least two content generation tasks.
3. Create one output writer task.

Run your code and verify reading, generation, and writing are all interleaved. Verify that running the pipeline with multiple generate workers is faster than running it with one.

## Going Further

* This pipeline only has I/O operations so `asyncio` works well. But what if the pipeline has CPU-heavy operations? For example, what if you need to split and count the words in the model responses? You can try using `loop.run_in_executor(executor, operation)` and `ProcessPoolExecutor` to offload CPU heavy work to a separate process and avoid blocking the event loop.

* In this challenge, we created a single reader and writer task since disk I/O is fast and the API calls were the bottleneck. However, if our reads/writes were slower (e.g., to a remote database or cloud storage), we could extend this pattern to have multiple concurrent reader/writer tasks.

```admonish success title=""
**Now take some time to attempt the challenge before looking at the solution!**
```

---

## Solution

Below is a walkthrough of one possible solution. Your implementation may differ, and that's okay!

First let's define all the imports and constants:

```python
import asyncio
import pathlib

import aiofiles
from google import genai

_INPUTS_PATH = "inputs/"
_OUTPUTS_PATH = "outputs/"
_GENERATE_TASKS = 5
_SHARD_SIZE = 5
_QUEUE_SIZE = 5

```

### Step 2 - Solution

```python
async def read_inputs(input_dir: pathlib.Path, input_queue: asyncio.Queue):
    for path in input_dir.iterdir():
        async with aiofiles.open(path) as f:
            async for prompt in f:
                if not prompt:
                    continue
                await input_queue.put(prompt)
                print(f"Enqueued prompt {prompt.strip()} from {path}")
```

### Step 3 - Solution

```python
async def generate_with_retry(
    client: genai.Client, input_queue: asyncio.Queue, output_queue: asyncio.Queue
):
    prompt = await input_queue.get()
    try:
        response = await client.aio.models.generate_content(
            model="gemini-flash-latest", contents=prompt
        )
        await output_queue.put((prompt, response.text))
        print(f"Generated content for prompt: {prompt.strip()}")
    except Exception as e:
        print(f"Error generating content for prompt {prompt.strip()}: {e}")
        # Re-queue the prompt for retry
        await input_queue.put(prompt)
    finally:
        input_queue.task_done()


async def generate_content(
    client: genai.Client, input_queue: asyncio.Queue, output_queue: asyncio.Queue
):
    try:
        while True:
            # Optionally wait between requests to avoid rate limiting.
            # await asyncio.sleep(0.15)
            await generate_with_retry(client, input_queue, output_queue)
    except asyncio.CancelledError:
        print("Shutting down content generation task...")
```

### Step 4 - Solution

```python
async def write_outputs(output_dir: pathlib.Path, output_queue: asyncio.Queue):
    try:
        # Create parent directories if they don't exist
        output_dir.mkdir(parents=True, exist_ok=True)
        shard_index = 0
        while True:
            path = output_dir / f"shard_{shard_index}.txt"
            async with aiofiles.open(path, "w") as f:
                for _ in range(_SHARD_SIZE):
                    prompt, response = await output_queue.get()
                    await f.write(f"{prompt} - {response} \n")
                    print(f"Wrote {prompt.strip()} response to {path}")
                    output_queue.task_done()
            shard_index += 1
    except asyncio.CancelledError:
        print("Shutting down writer...")
```

### Step 5 - Solution

```python
async def main():
    client = genai.Client()
    input_queue = asyncio.Queue(maxsize=_QUEUE_SIZE)
    output_queue = asyncio.Queue(maxsize=_QUEUE_SIZE)

    input_dir = pathlib.Path(_INPUTS_PATH)
    output_dir = pathlib.Path(_OUTPUTS_PATH)

    reader_task = asyncio.create_task(read_inputs(input_dir, input_queue))
    generate_tasks = [
        asyncio.create_task(generate_content(client, input_queue, output_queue))
        for _ in range(_GENERATE_TASKS)
    ]
    writer_task = asyncio.create_task(write_outputs(output_dir, output_queue))

    # Wait until all the inputs are read and processed
    await reader_task
    await input_queue.join()

    # Cancel the generate tasks since all inputs have been processed.
    for task in generate_tasks:
        task.cancel()
    await asyncio.gather(*generate_tasks, return_exceptions=True)

    # Wait until all outputs are written.
    await output_queue.join()
    writer_task.cancel()
    await writer_task


asyncio.run(main())
```

Now let's run this and check the output:

```bash
Enqueued prompt What is two times 27? from inputs/shard_9.txt
Enqueued prompt What is two times 28? from inputs/shard_9.txt
Enqueued prompt What is two times 29? from inputs/shard_9.txt
Enqueued prompt What is two times 6? from inputs/shard_2.txt
Generated content for prompt: What is two times 28?
Enqueued prompt What is two times 20? from inputs/shard_6.txt
Wrote What is two times 28? response to outputs/shard_0.txt
Generated content for prompt: What is two times 27?
Enqueued prompt What is two times 24? from inputs/shard_8.txt
Wrote What is two times 27? response to outputs/shard_0.txt
...
Generated content for prompt: What is two times 0?
Wrote What is two times 0? response to outputs/shard_0.txt
Generated content for prompt: What is two times 13?
Wrote What is two times 13? response to outputs/shard_0.txt
Generated content for prompt: What is two times 1?
Wrote What is two times 1? response to outputs/shard_0.txt
Generated content for prompt: What is two times 2?
Shutting down content generation task...
Shutting down content generation task...
Shutting down content generation task...
Wrote What is two times 2? response to outputs/shard_0.txt
Shutting down writer...
```

Note how the file reads, API calls, and file writes are all interleaved. As the producers (file reader, content generator) add items to the queues, the consumers (content generator, writer) get items from the queues and process them concurrently.

While the API is processing prompt #1, we can be reading prompt #2 from disk and writing the response for prompt #0. Without concurrency, each stage would wait idle for the others to complete.

With 1 generator task this script takes **~20 seconds** (sequential API calls), but with 5 generator tasks it takes **~6 seconds** (concurrent API calls). Multiple workers can process the slow stage (API calls) concurrently, while the fast stages (file I/O) keep them fed with work.
