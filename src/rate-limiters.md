# Rate Limiters

## Challenge

Model provider APIs have rate limits. It's very easy to exceed these limits if you're making requests concurrently without throttling.

In this challenge, you will build a [sliding window](https://medium.com/@avocadi/rate-limiter-sliding-window-log-44acf1b411b9) rate limiter for calling async APIs.

A sliding window rate limiter works by keeping a queue of the most recent `N` requests within a given time window. Before processing a new request, the rate limiter removes any requests that fall outside the time window, then checks if the number of remaining requests is below the limit. If so, the new request is added to the queue and processed. Otherwise, the limiter waits until old requests have aged out of the window.

### Step 0

To get started, get a Gemini API key from [Google AI Studio](https://aistudio.google.com/app/api-keys). We use the Gemini API because it has a generous free tier.

```bash
export GEMINI_API_KEY="YOUR_API_KEY"
```

### Step 1

In this step, your goal is to make concurrent requests to the Gemini API and hit the rate limits.

Create a new script (`script.py`) that makes 20 concurrent requests to the Gemini API then run your script. The solution to the [LLM Responses](llm-responses.md) chapter shows how to do this.

Confirm you get a resource exhausted error:

```bash
python script.py
> google.genai.errors.ClientError: 429 RESOURCE_EXHAUSTED. {'error': {'code': 429,
 'message': 'You exceeded your current quota, please check your plan and billing details.
  For more information on this error, head to: https://ai.google.dev/gemini-api/docs/rate-limits. ...
```

At the time of writing, Gemini 2.5 Flash had a limit fo 10 requests per minute on the [free tier](https://ai.google.dev/gemini-api/docs/rate-limits#current-rate-limits).

### Step 2

In this step, your goal is to implement the sliding window rate limiter.

Below is skeleton code for a `RateLimiter` class. You need to implement the `acquire()` method:

```python
import asyncio
from collections import deque
from datetime import timedelta


class RateLimiter:
    def __init__(self, limit: int, interval: timedelta):
        self._limit = limit
        self._interval = interval.total_seconds()
        # Holds request timestamps.
        self._window = deque()

    async def acquire(self) -> None:
        """Wait until a new request can be made under the rate limit."""
        pass
```

`acquire()` will be awaited until requests can be made under the rate limit:

```python
async def generate_content(index, client, rate_limiter):
    # Waits until we are under the rate limit.
    await rate_limiter.acquire()
    response = await client.aio.models.generate_content(
        model="gemini-flash-latest", contents="Why do some birds migrate?"
    )
    return response
```

The `acquire()` method should:

1. Use `asyncio.get_running_loop().time()` to get the current monotonic time in seconds.
2. Remove old requests from `window` to ensure it only has requests that were made within the past `interval` seconds.
3. If the window has fewer than `limit` requests, the request is allowed. Add the current request time to the `window` and return.
4. If the limit is reached, calculate how long to wait for the oldest request to age out of the window, then sleep with `asyncio.sleep()`.
5. Retry the above steps in a `while` loop.

```admonish hint title="Hint" collapsible=true
You can use `asyncio.Lock()` to prevent race conditions when making changes to the `window`.
```

### Step 3

In this step, your goal is to test your rate limiter.

Update your concurrent code to call `await limiter.acquire()` before making requests.

Verify that the rate limiter delays requests to avoid hitting the rate limit Gemini API rate limits.

### Going Further

* Try implementing other rate limiting algorithms like [token bucket](https://en.wikipedia.org/wiki/Token_bucket). This will require keeping track of "tokens" and replenishing them in every iteration at a fixed rate.
* Implement a sliding window rate limiter that avoids busy-waiting and respects request order. The rate limiter should **not** use `while` loops or `asyncio.sleep()`. When the limit is reached, create a Future with `loop.create_future()` and add it to a waiters queue, then await it. When a request is sent, use `loop.call_later(interval, callback)` to schedule a callback that will wake up the next waiter from the queue. Effectively, every allowed requests reserves a slot that expires in `interval` seconds and unblocks the next waiter in line.

```admonish success title=""
**Now take some time to attempt the challenge before looking at the solution!**
```

---

## Solution

Below is a walkthrough of one possible solution. Your implementation may differ, and that's okay!

### Step 1 - Solution

See the [LLM Responses](llm-responses.md) solution for how to make concurrent requests to the Gemini API. You can use that code but increase `_NUM_REQUESTS = 20` to trigger the rate limit error.

### Step 2 - Solution

```python
import asyncio
from collections import deque
from datetime import timedelta


class RateLimiter:
    def __init__(self, limit: int, interval: timedelta):
        self._limit = limit
        self._interval = interval.total_seconds()
        self._window = deque()
        self._lock = asyncio.Lock()

    def _prune_window(self, now: float) -> None:
        """Removes requests that have aged out of the time window."""
        while self._window and now - self._window[0] > self._interval:
            self._window.popleft()

    async def acquire(self) -> None:
        loop = asyncio.get_running_loop()
        while True:
            async with self._lock:
                now = loop.time()
                self._prune_window(now)

                if len(self._window) < self._limit:
                    # We have space in the sliding window to send a request.
                    self._window.append(now)
                    return

                # Wait for the oldest request to age out of the window.
                oldest_request_time = self._window[0]
                elapsed = now - oldest_request_time
                remaining = self._interval - elapsed

            await asyncio.sleep(remaining)
```

The key points:

* `_lock` prevents race conditions when multiple tasks call `acquire()` simultaneously
* `_prune_window()` removes requests outside the sliding window
* We release the lock before sleeping to allow other tasks to check the rate limit

Note that this solution suffers from the "thundering heard" problem. If multiple tasks are sleeping, all of them will wake up at the same time to try to acquire the lock. Only one request will be allowed, and the remaining tasks will need to sleep again.

One way to avoid this problem is to implement the rate limiter using futures as described in the [Going Further](#going-further) section.

### Step 3 - Solution

Now let's integrate the rate limiter with our Gemini API calls:

```python
import asyncio
from datetime import datetime, timedelta

from google import genai

_NUM_REQUESTS = 20


class RateLimiter:
    # Same as above.
    ...


async def generate_content(index, client, rate_limiter):
    await rate_limiter.acquire()
    print(f"Request {index} sent at {datetime.now().strftime('%H:%M:%S')}")
    response = await client.aio.models.generate_content(
        model="gemini-flash-latest", contents="Why do some birds migrate?"
    )
    return response


async def main():
    # Gemini Flash Latest has a rate limit of 10 requests per minute.
    limiter = RateLimiter(limit=10, interval=timedelta(minutes=1))

    client = genai.Client()
    tasks = [generate_content(i, client, limiter) for i in range(_NUM_REQUESTS)]
    results = await asyncio.gather(*tasks)


if __name__ == "__main__":
    asyncio.run(main())
```

Now when we run this with 20 requests, it completes successfully without hitting rate limits:

```bash
time python script.py
> Request 0 sent at 22:31:10
> Request 1 sent at 22:31:10
> Request 2 sent at 22:31:10
> Request 3 sent at 22:31:10
> Request 4 sent at 22:31:10
> Request 5 sent at 22:31:10
> Request 6 sent at 22:31:10
> Request 7 sent at 22:31:10
> Request 8 sent at 22:31:10
> Request 9 sent at 22:31:10
    # Note how we wait one minute before sending the 11th request to stay
    # within the rate limit.
> Request 19 sent at 22:32:10
> Request 18 sent at 22:32:10
> Request 17 sent at 22:32:10
> Request 16 sent at 22:32:10
> Request 15 sent at 22:32:10
> Request 14 sent at 22:32:10
> Request 12 sent at 22:32:10
> Request 11 sent at 22:32:10
> Request 13 sent at 22:32:10
> Request 10 sent at 22:32:10
> 
> real    1m9.061s
> user    0m2.072s
> sys     0m0.402s
```

The first 10 requests complete immediately, then the rate limiter automatically pauses until enough time has passed to send the next batch. All 20 requests succeed without any resource exhausted errors.
