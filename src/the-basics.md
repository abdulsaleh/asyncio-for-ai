# The Basics

There are many external guides on `asyncio` but many are confusing and go into too much detail. 


`asyncio` let's you run tasks concurrently. 



Best way is to learn by example. 

This guide assumes you have a basic understanding of `asyncio`. 



For example, let's say you're making requests to an external service or API, like the Gemini API. Each request can 

*concurrently*. 





### Concurrency != Parallelism

Note how I said *concurrently*, not in parallel. Technically, there is only one thread and one process. It's just that 




### Await, Create, and Gather

To start using `asyncio`, you need to learn about `await`, `create_task`, and `gather`.


`await` defines a point where a function can pause and yeild control for other tasks to execute.

```python



```