# Async Gemini API




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
