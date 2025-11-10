# The Basics


The chapter goes over just enough info to get you started with `asyncio`. 

The best way to learn and build a mental model of `asyncio` is by using it so this chapter will be light on the details. You can practice building things with `asyncio` in the following chapters and learn as you go along.


You can refer to this guide by Alexander Nordin if you want to learn in more detail.

https://github.com/anordin95/a-conceptual-overview-of-asyncio/tree/main


### Event Loops and Tasks


TODO: Explain what an event loop and task is. 

Use an example of making an external API call. instead of making them sequentially, you can make them concurrently. 

Don't include code yet, just conceptual.



### Await, Create, and Gather

TODO: explain how to use await, create, and gather. Write code for the above example

```python

```




### Synchronization Primitives

There are a few ways to co-ordinate behvior between tasks. The simplest way is locks (to protect a shared resource), but there are other options as well.

E.g. let's say you want to check and update an element in a shared dictionary.

https://docs.python.org/3/library/asyncio-sync.html





### When to use asyncio

`asyncio` is best used for I/O bound tasks, not CPU bound tasks. I/O bound tasks include ...

The Python Global Interpreter Lock (GIL) prevents you from doing work in parallel. 

If you're doing CPU heavy work, then you should use multi-processing instead. 

It's also possible to use asyncio in combination with multiprocessing if you're doing work that requires both I/O and CPU processing. For example, you might be making some API calls (I/O) then processing the result (CPU). The [Data Pipelines](data-pipelines.md) chapter covers this in more detail.


### Going Further

There are many other concepts. But you will encounter them throughout the guide. It will be easier to understand them in context of the future chapters and motivating examples. 