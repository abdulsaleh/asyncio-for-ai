# Web Crawlers

## Challenge

In this challenge, you will build a concurrent web crawler with `asyncio`. AI models often train on data scraped from the web so efficient web crawlers are important for collecting training data at scale.

Instead of fetching pages one at a time, the crawler processes multiple requests concurrently. This is more efficient because most of the time is spent waiting for responses so it can happen concurrently.

The crawler will:

1. Maintain a queue of URLs to visit
2. Use multiple worker tasks to fetch and parse pages concurrently
3. Track visited URLs to avoid duplicates, extract links from each page, and add new URLs to the queue
4. Output parsed pages to a separate queue for downstream processing

## Before you start

The following functions or classes are relevant for this chapter. It might be helpful to read their docs before you start:

* `asyncio.Queue()` for managing the crawl queue
* `asyncio.Lock()` for protecting shared state
* `asyncio.create_task()` for spawning worker tasks
* `task.cancel()` for gracefully shutting down workers
* `queue.task_done()` and `queue.join()` for tracking completion
* `aiohttp.ClientSession()` for making async HTTP requests
* `BeautifulSoup` from `bs4` for parsing HTML

### Step 0

To get started, install the required packages:

```bash
pip install aiohttp beautifulsoup4
```

We'll use [web-scraping.dev](https://web-scraping.dev/products) (`https://web-scraping.dev/products`) since it's designed for web scraping practice. But you can also use this web crawler for scraping other sites (e.g. Wikipedia).

### Step 1

In this step, your goal is to implement a coroutine that fetches and parses a web page.

First, create a `ParsedPage` dataclass to hold the results:

```python
from dataclasses import dataclass

@dataclass
class ParsedPage:
    """Parsed page with html and child URLs found on that page."""
    url: str
    html: str
    child_urls: set[str]
```

Then implement the fetch and parse function:

```python
import aiohttp

async def fetch_and_parse(url: str, session: aiohttp.ClientSession) -> ParsedPage:
    """Fetch a page and extract links."""
    pass
```

This function should:

1. Fetch the page using `session.get(url)`.
2. Get the HTML text with `await response.text()`
3. Extract the child urls that appear on that page.
4. Return a `ParsedPage` with the URL, HTML, and child URLs.

```admonish warning title="Important"
Always reuse the same `aiohttp.ClientSession` across requests. Creating a new session for each request is slow and wastes resources. Create the session once and pass it to all worker tasks.
```

This chapter is focused on `asyncio` patterns, not web parsing techniques, so you can just use the below helper function for extracting the child urls:

```python
from urllib.parse import urljoin, urlparse
from bs4 import BeautifulSoup

def extract_urls(html: str, url: str) -> set[str]:
    """Extracts URLs from the given page."""
    soup = BeautifulSoup(html, "html.parser")
    parsed_base = urlparse(url)
    base_domain = parsed_base.netloc
    child_urls = set()

    for anchor in soup.find_all("a", href=True):
        href = str(anchor["href"])
        absolute_url = urljoin(url, href)
        parsed = urlparse(absolute_url)

        # Only keep URLs from the same domain
        if parsed.netloc != base_domain:
            continue

        child_urls.add(absolute_url)

    return child_urls
```

### Step 3

In this step, your goal is to implement the `WebCrawler` class with its worker tasks.

The crawler uses breadth-first search, so it needs to maintain a queue of urls to visit, a set of visited urls, and a lock to protect the set:

```python
import asyncio

class WebCrawler:
    def __init__(self, seed_urls: list[str], max_pages: int):
        self._max_pages = max_pages
        self._crawl_queue = asyncio.Queue()
        self._visited = set()
        self._lock = asyncio.Lock()

        # Add seed URLs to the queue and mark as visited
        for url in seed_urls:
            self._crawl_queue.put_nowait(url)
            self._visited.add(url)

    async def _crawl_task(
        self,
        parsed_queue: asyncio.Queue[ParsedPage | None],
        session: aiohttp.ClientSession,
    ) -> None:
        pass

    async def crawl(
        self, num_workers: int, parsed_queue: asyncio.Queue[ParsedPage | None]
    ) -> None:
        pass
```

We initialize the crawl queue and visited set with the seed urls. The worker tasks can then start reading pages from the crawl queue.

#### Implementing the worker task

The `_crawl_task()` method is what actually executes the breadth-first search. Each crawl task:

1. Gets a URL from the crawl queue
2. Fetches and parses the page
3. Adds the parsed result to the output `parsed_queue`
4. Extracts child URLs and adds new ones to the crawl queue

The crawl task gets new urls from the crawl queue in a loop until the queue is empty. We should stop adding new pages after we've visited `max_pages`.

#### Implementing the main crawl method

The `crawl()` method should:

1. Initialize the shared client session.
2. Spawn multiple worker tasks
3. Wait for the crawl queue to be empty
4. Cancel all workers gracefully

### Step 4

In this step, your goal is to implement the output logger.

The output logger reads parsed pages from the output queue and processes them. In a real system, you might save the HTML to disk or a database, but for this challenge, we'll just print the results.

```python
async def log_results(parsed_queue: asyncio.Queue[ParsedPage | None]):
    pass
```

### Step 5

Now tie everything together in the main function. Create and run the logger and crawler tasks.

```python
async def main():
    urls = ["https://web-scraping.dev/products"]
    crawler = WebCrawler(seed_urls=urls, max_pages=1000)

    parsed_queue = asyncio.Queue()
    log_task = asyncio.create_task(log_results(parsed_queue))

    await crawler.crawl(num_workers=10, parsed_queue=parsed_queue)

    # Optionally pass sentinel value to shut down logger.
    await parsed_queue.put(None)
    await log_task

asyncio.run(main())
```

Run your crawler and verify that:

* Multiple pages are fetched concurrently
* Duplicate URLs are not visited
* The crawler stops after reaching the max page limit or visiting all links
* Using 1 worker is slower than using multiple.

## Going Further

* Add a `max_depth` limit to control the depth of the search.

* Add request throttling per domain to be polite to servers. Use the rate limiter from the [Rate Limiters](rate-limiters.md) chapter. For an added challenge, use separate rate limiters for each domain.

* The `extract_urls` function call and `BeautifulSoup` parsing is *not* async so it blocks the event loop. Try using `loop.run_in_executor()` to extract the URLs. Does it affect performance? Why or why not?

```admonish success title=""
**Now take some time to attempt the challenge before looking at the solution!**
```

---

## Solution

Below is a walkthrough of one possible solution. Your implementation may differ, and that's okay!

First let's define all the imports and constants:

```python
import asyncio
from dataclasses import dataclass
from urllib.parse import urljoin, urlparse

import aiohttp
from bs4 import BeautifulSoup

_NUM_WORKERS = 10
_MAX_PAGES = 1000
```

### Step 1 - Solution

```python
@dataclass
class ParsedPage:
    """Parsed page with html and child URLs found on that page."""

    url: str
    html: str
    child_urls: set[str]


def extract_urls(html: str, url: str) -> set[str]:
    """Extracts URLs from the given page."""
    soup = BeautifulSoup(html, "html.parser")
    parsed_base = urlparse(url)
    base_domain = parsed_base.netloc
    child_urls = set()

    for anchor in soup.find_all("a", href=True):
        href = str(anchor["href"])
        absolute_url = urljoin(url, href)
        parsed = urlparse(absolute_url)

        # Only keep URLs from the same domain
        if parsed.netloc != base_domain:
            continue

        child_urls.add(absolute_url)

    return child_urls


def parse_page(html: str, url: str) -> ParsedPage:
    """Extract links from HTML and filter to same-domain URLs."""
    return ParsedPage(
        url=url,
        html=html,
        child_urls=extract_urls(html, url),
    )


async def fetch_and_parse(url: str, session: aiohttp.ClientSession) -> ParsedPage:
    """Fetch a page and extract links."""
    async with session.get(url) as response:
        html = await response.text()
        parsed_page = parse_page(html, url)
        return parsed_page
```

### Step 3 - Solution

```python
class WebCrawler:
    def __init__(self, seed_urls: list[str], max_pages: int):
        self._max_pages = max_pages

        self._crawl_queue = asyncio.Queue()
        self._visited = set()
        self._lock = asyncio.Lock()

        for url in seed_urls:
            self._crawl_queue.put_nowait(url)
            self._visited.add(url)

    async def crawl(
        self, num_workers: int, parsed_queue: asyncio.Queue[ParsedPage | None]
    ) -> None:
        async with aiohttp.ClientSession() as session:
            tasks = [
                asyncio.create_task(self._crawl_task(parsed_queue, session))
                for _ in range(num_workers)
            ]
            await self._wait_until_done(tasks)

    async def _wait_until_done(self, crawl_tasks: list[asyncio.Task]) -> None:
        """Waits for crawling to complete and shuts down tasks."""
        await self._crawl_queue.join()
        for task in crawl_tasks:
            task.cancel()
        await asyncio.gather(*crawl_tasks, return_exceptions=True)

    async def _crawl_task(
        self,
        parsed_queue: asyncio.Queue[ParsedPage | None],
        session: aiohttp.ClientSession,
    ) -> None:
        while True:
            try:
                url = await self._crawl_queue.get()
            except asyncio.CancelledError:
                print("Shutting down crawl task...")
                return

            # Parse the page and add it to the output queue.
            try:
                parsed_page = await fetch_and_parse(url, session)
                await parsed_queue.put(parsed_page)
                print(f"Parsed {parsed_page.url}.")

                async with self._lock:
                    for url in parsed_page.child_urls:
                        if url in self._visited:
                            continue
                        if len(self._visited) >= self._max_pages:
                            break
                        self._visited.add(url)
                        await self._crawl_queue.put(url)

            except Exception as e:
                print(f"Error fetching {url}: {e}")

            finally:
                self._crawl_queue.task_done()
```

The `crawl()` method creates an `aiohttp.ClientSession` that's shared across all workers. This enables connection pooling and is more efficient than creating a new session per request.

The `_crawl_task()` method implements the breadth-first search. Note that:

* The `while True` loop runs forever until the task is cancelled.
* We catch `asyncio.CancelledError` when getting from the queue to handle graceful shutdown. See the [Data Pipelines](data-pipelines.md) chapter for a similar example of producer-consumer queues.
* The lock protects the visited set. Multiple workers could try to add the same URL simultaneously without the lock.
* We check the max pages limit inside the lock to ensure we don't exceed it.
* The `finally` block ensures `task_done()` is called even if an error occurs, which is important so `queue.join()` can work correctly.

The `_wait_until_done()` method waits for the queue to be empty with `queue.join()`, then cancels all workers.

### Step 4 - Solution

```python
async def log_results(parsed_queue: asyncio.Queue[ParsedPage | None]) -> None:
    while True:
        parsed_page = await parsed_queue.get()

        if parsed_page is None:
            parsed_queue.task_done()
            print("Shutting down writer...")
            return

        print(f"Logged {parsed_page.url} with {len(parsed_page.html)} chars.")

        parsed_queue.task_done()
```

### Step 5 - Solution

```python
async def main():
    urls = [
        "https://web-scraping.dev/products",
    ]
    crawler = WebCrawler(seed_urls=urls, max_pages=_MAX_PAGES)

    parsed_queue = asyncio.Queue()
    log_task = asyncio.create_task(log_results(parsed_queue))

    await crawler.crawl(num_workers=_NUM_WORKERS, parsed_queue=parsed_queue)

    # Stop writer after all crawlers complete.
    await parsed_queue.put(None)
    await log_task


asyncio.run(main())
```

Now let's run this and check the output:

```bash
python script.py

...
Parsed https://web-scraping.dev/products?category=household.
Parsed https://web-scraping.dev/products?category=consumables.
Parsed https://web-scraping.dev/cart.
Parsed https://web-scraping.dev/testimonials.
Logged https://web-scraping.dev/sitemap.xml with 6689 chars.
Logged https://web-scraping.dev/file-download with 19476 chars.
Logged https://web-scraping.dev/products?category=household with 12582 chars.
Logged https://web-scraping.dev/products?category=consumables with 17133 chars.
Logged https://web-scraping.dev/cart with 11856 chars.
Logged https://web-scraping.dev/testimonials with 29975 chars.
Parsed https://web-scraping.dev/products?page=5.
Parsed https://web-scraping.dev/login.
Parsed https://web-scraping.dev/products?page=2.
Parsed https://web-scraping.dev/product/3.
Parsed https://web-scraping.dev/product/5.
...
Logged https://web-scraping.dev/products?category=household&page=3 with 8848 chars.
Parsed https://web-scraping.dev/products?category=consumables&page=5.
Logged https://web-scraping.dev/products?category=consumables&page=5 with 8848 chars.
Shutting down crawl task...
Shutting down crawl task...
Shutting down crawl task...
Shutting down crawl task...
Shutting down crawl task...
Shutting down crawl task...
Shutting down crawl task...
Shutting down crawl task...
Shutting down crawl task...
Shutting down crawl task...
Shutting down writer...
```

Note how:

* Multiple pages are fetched and logged concurrently.
* All workers shut down gracefully when the queue is empty.

This script takes about 12s to run with 1 worker and 4s to run with 5 workers as pages are fetched concurrently.
