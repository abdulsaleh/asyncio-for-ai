# The Basics

This chapter covers just enough to get you started with `asyncio`.

The best way to learn is by using it. This chapter will be light on details. You'll practice in the following chapters and learn as you go.

For deeper understanding, refer to [Alexander Nordin's conceptual overview](https://github.com/anordin95/a-conceptual-overview-of-asyncio/tree/main).

## Event Loops and Tasks

When you make an API call, your program waits. The network is slow. Your CPU sits idle.

With `asyncio`, you don't wait. While one API call is in flight, you can start another. When any call completes, your program handles it. This is **concurrency**.

**The event loop** is what makes this work. It keeps track of all your tasks and switches between them. When a task is waiting on I/O (like an API call), the event loop gives control to another task.

**A task** is a unit of work. Making an API call is a task. Reading a file is a task. Tasks can pause and resume without blocking each other.

Say you need to call an API three times. Sequentially, this might take 3 seconds (1 second per call). With `asyncio`, all three calls can be in flight at once. Total time: 1 second.

That's the core idea. The rest is syntax.

## Await, Create, and Gather

Here's how you write concurrent code with `asyncio`.

**`await`** pauses a task until an operation completes. It gives control back to the event loop so other tasks can run.

```python
import asyncio
import httpx

async def fetch_url(url: str):
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
        return response.text

# This function pauses at 'await' until the HTTP request completes
```

**`asyncio.create_task()`** schedules a task to run concurrently. The task starts running immediately without waiting for it to finish.

```python
async def main():
    # These start running concurrently
    task1 = asyncio.create_task(fetch_url("https://example.com"))
    task2 = asyncio.create_task(fetch_url("https://example.org"))

    # Wait for both to complete
    result1 = await task1
    result2 = await task2
```

**`asyncio.gather()`** runs multiple tasks concurrently and waits for all of them to complete.

```python
async def main():
    urls = [
        "https://example.com",
        "https://example.org",
        "https://example.net"
    ]

    tasks = [fetch_url(url) for url in urls]
    results = await asyncio.gather(*tasks)
    # All three requests run concurrently
```

To run your async code, use `asyncio.run()`:

```python
if __name__ == "__main__":
    asyncio.run(main())
```

That's it. `async def` to define async functions. `await` to pause. `gather()` to run things concurrently. `asyncio.run()` to start the event loop.




## Synchronization Primitives

Sometimes tasks need to coordinate. A shared resource can't be modified by two tasks at once. That's where synchronization primitives help.

**`asyncio.Lock`** ensures only one task accesses a resource at a time. Say you have multiple tasks updating a shared dictionary:

```python
import asyncio

counter = {"count": 0}
lock = asyncio.Lock()

async def increment():
    async with lock:
        # Only one task can be here at a time
        current = counter["count"]
        await asyncio.sleep(0.01)  # Simulate some work
        counter["count"] = current + 1

async def main():
    await asyncio.gather(*[increment() for _ in range(100)])
    print(counter["count"])  # Prints 100

asyncio.run(main())
```

Without the lock, you'd get race conditions and unpredictable results.

Other primitives exist: `Event`, `Semaphore`, `Condition`, and `Queue`. You'll see `Queue` in the [Data Pipelines](data-pipelines.md) chapter. For the rest, see the [official docs](https://docs.python.org/3/library/asyncio-sync.html).





## When to Use Asyncio

Use `asyncio` for I/O-bound work. API calls. Database queries. File operations. Reading from disk. Anything that waits on something external.

Don't use `asyncio` for CPU-bound work. Heavy computation. Image processing. Number crunching. The Python Global Interpreter Lock (GIL) prevents true parallelism, so you won't get speedups.

For CPU-heavy tasks, use `multiprocessing` instead.

You can combine both. Make API calls with `asyncio`, then offload CPU work to a process pool. The [Data Pipelines](data-pipelines.md) chapter shows how.


## Going Further

There's more to learn. Error handling. Timeouts. Cancellation. Context managers. But you don't need it all upfront.

You'll encounter these concepts throughout the guide. They're easier to understand with real examples and motivation.

Now let's build something. Start with [Async LLM Responses](llm-responses.md). 