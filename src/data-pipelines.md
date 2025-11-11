# Data Pipelines

## Challenge

In this challenge, you will build a concurrent data pipeline. The pipeline will read some prompts from disk, call the Gemini API, then write the model responses back to disk.

Reading/writing files and sending API requests are I/O operations so `asyncio` is a great fit for speeding up this pipeline.

You can also extend this pipeline to do CPU heavy tasks. See the [Going Further](#going-further) section if you want to learn how.

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

Note how the producer and consumer steps are **interleaved**. Items are processed as they are added. This is the beauty of concurrent code.

```admonish warning title="Important"
The consumers loop forever waiting on new items. You need to cancel the consumers explicitly after the producers are done and the queue is empty.
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

---

The data pipeline in this challenge will have three tasks:

1. A prompt reader task that reads the input prompts from disk and adds them to the prompt queue.
2. A generate content task that reads from the prompt queue and adds them to the responses queue.
3. A response writer task that reads from the responses queue and writes the responses to disk.


### Step 0

Get a gemini API key.


### Step 1

Call the Gemini API concurrently.


### Step 2

In this step, your goal is to implement the async reader function (coroutine).

The function takes the path to a file, reads the contents, and adds then to the prompt queue.

Each line in the file is a separate prompt.

You can generate the prompt files using this command:
```bash

```

```python
async def read_prompts(path: pathlib.Path, prompt_queue: asyncio.Queue):
    pass
```


### Step 3

In this step, your goal is to implement the async generate content function.

This function reads prompts from the prompt queue, makes requests to the Gemini API, then writes the responses to the response queue.

```python
async def generate_content(client: genai.Client, prompt_queue: asyncio.Queue, response_queue: asyncio.Queue)
    pass
```

```admonish tip title="Tip"
To avoid hitting rate limits, you can sleep between requests or you can use the rate limiter we implemented in the [Rate Limiters](#rate-limiters.md) chapter.
```

### Step 4

In this step, your goal is to implement the async writer function.

This function reads the responses from the response queue, then writes them to the given output file path.

```python
async def write_response(path: pathlib.Path, response_queue: asyncio.Queue):
    return 
```

### Step 5

Create multiple readers, 



## Going Further

* This pipeline only has I/O operations so `asyncio` is a natural fit. But what if the pipeline has CPU-heavy operations? For example, what if you need to split and count the words in the model responses? You can try using `loop.run_in_executor(executor, operation)` and `ProcessPoolExecutor` to offload CPU heavy work to a separate process and avoid blocking the event loop.

```admonish success title=""
**Now take some time to attempt the challenge before looking at the solution!**
```

---

## Solution

Below is a walkthrough of one possible solution. Your implementation may differ, and that's okay!


TODO: add solution