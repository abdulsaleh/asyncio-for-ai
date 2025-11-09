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

At the time of writing, Gemini 2.5 Flash had a limit fo 10 requests per minute on the [free tier](https://ai.google.dev/gemini-api/docs/rate-limits#current-rate-limits):

| Model | RPM | TPM | RPD |
|-------|-----|-----|-----|
| Gemini 2.5 Pro | 2 | 125,000 | 50 |
| Gemini 2.5 Flash | 10 | 250,000 | 250 |

Create a new script (`script.py`) that makes 20 concurrent requests to the Gemini API then run your script. See the [LLM Responses](llm-responses.md) chapter to learn how to do this.

You should get a resource exhausted error:

```bash
python script.py
> google.genai.errors.ClientError: 429 RESOURCE_EXHAUSTED. {'error': {'code': 429,
 'message': 'You exceeded your current quota, please check your plan and billing details.
  For more information on this error, head to: https://ai.google.dev/gemini-api/docs/rate-limits. ...
```

### Step 2



```python
import asyncio
from collections import deque
from datetime import timedelta


class RateLimiter:
    def __init__(self, limit: int, window: timedelta):
        self._limit = limit
        self._window = window.total_seconds()
        self._history = deque()

    async def acquire(self) -> None:
        """Wait until a new request can be made under the rate limit."""
        pass
```

### Step 3

In this step, your goal is to test your rate limiter.

Update your concurrent code to call `await limiter.acquire()` before making requests and verify you no longer hit the Gemini API rate limits. 


## Solution


### Step 1 - Solution


### Step 2 - Solution 

```python
import asyncio
from collections import deque
from datetime import timedelta


class RateLimiter:
    def __init__(self, limit: int, window: timedelta):
        self._limit = limit
        self._window = window.total_seconds()
        self._history = deque()
        self._lock = asyncio.Lock()

    def _prune_history(self, now: float) -> None:
        """Removes requests that have aged out of the time window."""
        while self._history and now - self._history[0] > self._window:
            self._history.popleft()

    async def acquire(self) -> None:
        loop = asyncio.get_event_loop()
        while True:
            async with self._lock:
                now = loop.time()
                self._prune_history(now)

                if len(self._history) < self._limit:
                    # We have space in the sliding window to send a request.
                    self._history.append(now)
                    return

                # Wait for the oldest request to age out of the window.
                oldest_request_time = self._history[0]
                elapsed = now - oldest_request_time
                remaining = self._window - elapsed

            await asyncio.sleep(remaining)

```