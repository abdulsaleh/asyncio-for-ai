# Data Pipelines

## Challenge

In this challenge, you will build a concurrent data pipeline. The pipeline will read some prompts from disk, call the Gemini API, then write the model responses back to disk.

Reading/writing files and sending API requests are I/O operations so `asyncio` is a great fit for speeding up this pipeline. Instead of waiting on disk I/O or model responses, you can be doing work concurrently to save time.

You can also extend this pipeline to do CPU-bound work. See the [Going Further](#going-further) section if you want to learn how.

## Producer-Consumer Queues

You will be using a producer-consumer queue in this challenge, which is a key design pattern for concurrent code.

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

Note how the producer and consumer steps are **interleaved**. Items are processed as they are added. This is the benefit of concurrent code.

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

```admonish warning title="Important"
The consumers loop forever waiting on new items. You need to cancel the consumers explicitly after the producers are done and the queue is empty.
```

---
---

The data pipeline in this challenge will have:

1. **Input reader** tasks which read input prompts from disk then add them to the input queue.
2. **Content generation** tasks which get prompts from the input queue, call the Gemini API, then add responses to the output queue.
3. **Output writer** tasks which get the responses from the outputs queue then write them to disk.

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

The input reader coroutine takes the path to a file, reads the contents, and adds them to the input queue. Each line in the file is a separate prompt.

Use `aiofiles` for async file operations.

```python
import aiofiles

async def read_inputs(path: pathlib.Path, input_queue: asyncio.Queue):
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

1. It creates an output file or shard at the output directory.
2. It gets responses from the output queue, then writes them to the output file.
3. If the shard has `_SHARD_SIZE` responses, we close out that shard and create a new one.

```python
_SHARD_SIZE = 5

async def write_outputs(
    output_dir: pathlib.Path, output_queue: asyncio.Queue, shard_lock: asyncio.Lock
):
    return 
```

```admonish warning title="Important"
Make sure there is only one task writing to any given file at a time. Having multiple tasks write to the same file or shard can cause race conditions.
```

### Step 5

In this step, your goal is to create the final consumer-producer pipeline, chaining all the coroutines you have implemented above.

You need to define:

1. The input queue.
2. The output queue.
3. The input paths.
4. The shard lock.

You then need to create the tasks or workers:

1. Create one reader task per input file.
2. Create at least two content generation tasks.
3. Create at least two writer tasks.

Remember to cancel the consumer tasks after the input queues are empty.

Run your code and verify it works.

## Going Further

* This pipeline only has I/O operations so `asyncio` is a natural fit. But what if the pipeline has CPU-heavy operations? For example, what if you need to split and count the words in the model responses? You can try using `loop.run_in_executor(executor, operation)` and `ProcessPoolExecutor` to offload CPU heavy work to a separate process and avoid blocking the event loop.

* In the this challenge, we created a different reader task for each file. This works when the number of files is small, but what if we have more files? Creating too many reader tasks can be inefficient. One option is to add the file names to a queue and then update the reader task to process the file names one-by-one from that queue. That way we can have a small number of reader tasks processing a large number of files.

* Above we used the number of files in the output directory to get the shard index, `index = len(list(output_dir.glob("shard_*.txt")))`. This can be slow if we have a large number of shards. Another option is to keep a shard counter protected by a lock (e.g., `shard_counter = {'index': 0})`). We can pass this counter to the writers to increment and create the correct shard id.

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
_GENERATE_TASKS = 2
_WRITER_TASKS = 3
_SHARD_SIZE = 5
```

### Step 2 - Solution

```python
async def read_inputs(path: pathlib.Path, input_queue: asyncio.Queue):
    async with aiofiles.open(path) as f:
        async for prompt in f:
            if not prompt:
                continue
            await input_queue.put(prompt)
            print(f"Enqueued prompt {prompt.strip()} from {path}")
            # Simulate I/O delay. Yield control to event loop.
            await asyncio.sleep(1)
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
            # Wait between requests to avoid rate limiting.
            await asyncio.sleep(0.15)
            await generate_with_retry(client, input_queue, output_queue)
    except asyncio.CancelledError:
        print("Shutting down content generation task...")
```

### Step 4 - Solution

```python
async def create_shard(
    output_dir: pathlib.Path, shard_lock: asyncio.Lock
) -> pathlib.Path:
    async with shard_lock:
        index = len(list(output_dir.glob()))
        path = output_dir / f"shard_{index}.txt"
        path.touch(exist_ok=False)
    return path


async def write_outputs(
    output_dir: pathlib.Path, output_queue: asyncio.Queue, shard_lock: asyncio.Lock
):
    # Create parent directories if they don't exist
    output_dir.mkdir(parents=True, exist_ok=True)
    try:
        while True:
            # Create a new shard file at the next available index
            path = await create_shard(output_dir, shard_lock)
            async with aiofiles.open(path, "w") as f:
                for _ in range(_SHARD_SIZE):
                    prompt, response = await output_queue.get()
                    await f.write(f"{prompt} - {response} \n")
                    print(f"Wrote {prompt.strip()} response to {path}")
                    output_queue.task_done()

    except asyncio.CancelledError:
        print("Shutting down writer...")
```

### Step 5 - Solution

```python
async def main():
    client = genai.Client()
    input_queue = asyncio.Queue()
    output_queue = asyncio.Queue()
    shard_lock = asyncio.Lock()

    input_dir = pathlib.Path(_INPUTS_PATH)
    output_dir = pathlib.Path(_OUTPUTS_PATH)
    input_paths = [pathlib.Path(p) for p in input_dir.iterdir()]

    readers_tasks = [
        asyncio.create_task(read_inputs(p, input_queue)) for p in input_paths
    ]
    generate_tasks = [
        asyncio.create_task(generate_content(client, input_queue, output_queue))
        for _ in range(_GENERATE_TASKS)
    ]
    writer_tasks = [
        asyncio.create_task(write_outputs(output_dir, output_queue, shard_lock))
        for _ in range(_WRITER_TASKS)
    ]

    # Wait until all the inputs are read and processed
    await asyncio.gather(*readers_tasks)
    await input_queue.join()

    # Cancel the generate tasks since all inputs have been processed.
    for task in generate_tasks:
        task.cancel()
    await asyncio.gather(*generate_tasks, return_exceptions=True)

    # Wait until all outputs are written.
    await output_queue.join()
    for task in writer_tasks:
        task.cancel()
    await asyncio.gather(*writer_tasks)


asyncio.run(main())
```

Now let's run this and check the output:

```bash
> Enqueued prompt What is two times 15? from inputs/shard_5.txt
> Enqueued prompt What is two times 3? from inputs/shard_1.txt
> ...
> Generated content for prompt: What is two times 6?
> Enqueued prompt What is two times 28? from inputs/shard_9.txt
> Enqueued prompt What is two times 7? from inputs/shard_2.txt
> Generated content for prompt: What is two times 27?
> Wrote What is two times 27? response to outputs/shard_1.txt
> Generated content for prompt: What is two times 21?
> Wrote What is two times 21? response to outputs/shard_2.txt
> Generated content for prompt: What is two times 18?
> Wrote What is two times 18? response to outputs/shard_0.txt
> Enqueued prompt What is two times 11? from inputs/shard_3.txt
> ...
> Wrote What is two times 11? response to outputs/shard_4.txt
> Generated content for prompt: What is two times 2?
> Shutting down content generation task...
> Shutting down content generation task...
> Wrote What is two times 2? response to outputs/shard_5.txt
> Shutting down writer...
> Shutting down writer...
> Shutting down writer...
```

Note how the file reads, API calls, and file writes are all interleaved. As the producers (file reader, content generator) add items to the producer queue, the consumers (content generator, writer) get items from the queue and process them concurrently.
