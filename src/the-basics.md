# The Basics

## Coroutines and Event Loops

`asyncio`

* Coroutines: A coroutine is like a task. It runs concurrently and is non-blocking.

* Event Loops: The event loop manages multiple coroutines. It switches between coroutines to run them concurrently.

## Create, Await, and Gather

```
```

## Async Calls are Non-Blocking

```python
import asyncio 

_NUM_CALLS = 10
_DELAY_SECONDS = 1

async def send_request():
    """Simulates sending a request and waiting on the response."""
    await asyncio.sleep(_DELAY_SECONDS)


async def main():
    await asyncio.gather(*[send_request() for _ in range(_NUM_CALLS)])


asyncio.run(main())
```

This code takes roughly one second to execute. It makes ten calls, but the calls
are non blocking. `await asyncio.sleep(...)` passes control back to the
event loop so other calls can be made in concurrently.

Here, `asyncio.sleep(...)` simulates an I/O bound operation like sending a request,
querying a database, or reading a file from disk.

## Producer-Consumer Pipelines

Producer-Consumer pipelines are a common pattern for using `asyncio`.

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

In this pipeline, both the producer and consumer are running concurrently. As the
producer is adding items to the queue, the consumer is running and processing the
items.

A producer-consumer pipeline has two steps:

1. Setup:
    * A producer adds items to queue.
    * A consumer gets items from the queue and marks them as done.
2. Cleanup:
    * Wait until the producer is done and the queue is empty.
    * Cancel the consumer tasks.

We need to explicitly cancel the consumers, otherwise they will keep waiting on queue
items (`queue.get()`) indefinitely.

Another option is to use a "poison pill" by adding `None` to the queue which signals to
the consumer to break out of the `while` loop and terminate.

## Concurrency vs Parallelism
