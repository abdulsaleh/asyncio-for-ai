# Async LLM Responses


## Challenge

This challenge is to build an asynchronous LLM client. 



### Step 0

Get a Gemini API key. We use the Gemini API here because it has a generous free tier. 


### Step 1

```
from google import genai

client = genai.Client(api_key="YOUR_API_KEY")

response = await client.aio.models.generate_content(
    model='gemini-flash-latest',
    contents='Tell me a story in 300 words.'
)
```


### Step 2




### Step 3


```
from google import genai

client = genai.Client()

response = await client.aio.models.generate_content(
    model='gemini-flash-latest',
    contents='Tell me a story in 300 words.'
)
```


### Open Questions
* What happens when you call 100 concurrently? At what point do you get throttled? 
* 


## Solution







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
