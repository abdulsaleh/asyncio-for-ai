# Rate Limiters

## Challenge

Model provider APIs have rate limits. It's very easy to exceed these limits if you're making requests concurrently without throttling. In this challenge, you will build a [sliding window](https://medium.com/@avocadi/rate-limiter-sliding-window-log-44acf1b411b9) rate limiter for calling async APIs.

Let's say you have a rate limit of 10 requests per second. A sliding window rate limiter works by keeping a queue of the most recent 10 request timestamps.

Before processing a new request, the limiter checks the timestamp of the oldest request in the queue. If that timestamp is at least 1 second old, it means the oldest request has "aged out" of the time window, so the new request can proceed immediately. Otherwise, the limiter waits until enough time has passed for the oldest request to age out of the 1-second window.

This challenge will walk you through the implementation.

### Step 0

Get a Gemini API key from [Google AI Studio](https://aistudio.google.com/app/api-keys). We use the Gemini API because it has a generous free tier.

```bash
export GEMINI_API_KEY="YOUR_API_KEY"
```

### Step 1

In this step, your goal is to make concurrent requests to the Gemini API and hit the rate limits.

At the time of writing, these were the free tier [rate limits](https://ai.google.dev/gemini-api/docs/rate-limits#current-rate-limits) for the Gemini API:

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



```


```

## Solution
