# Celery and Asynchronous Processing in Python

## What is celery?

Celery is a distributed task queue framework that allows us to run tasks asynchronously in a separate worker process. It is designed to handle a large number of tasks, providing robust, flexible, and efficient task management. Celery is used to offload time-consuming tasks from the main application process, allowing for better performance and responsiveness.

In addition to being a task queue, we can configure celery as a task scheduler, i.e, periodically run some tasks

Another concept I would like to introduce here is worker nodes. They are the actual processes which execute the tasks which we send to the queue. Celery uses message brokers like RabbitMQ, Redis and AmazonSQS to handle communication between the worker nodes and the actual scheduler.

By default, celery starts in the prefork mode for concurrent execution of tasks, with the defualt concurrency equal to the number of CPU cores available on the machine. This can be changed using the `--concurrency` flag while starting celery. In this prefork method, each worker divides into multiple child process to achieve concurrency.This is based on python's multiprocessing package, and it helps avoid the Global Interpreter Lock (GIL) of python. It is good for CPU intensive tasks, but it is expensive to run.

To start celery,

```
celery -A celery_config_module worker
```

`--pool=threads` uses the ThreadPoolExecutor. With threads, celery uses actual operating system threads. In this approach, the GIL still applies, thus limiting the speed of our processing.

Another option to run celery is to use `--pool=solo`, This is a blocking but fast way to run our celery worker. By blocking, I mean that once a task starts, it runs till completion, and no other tasks run in the meantime. This is also faster since there are no overheads of context switching which come with threads or prefork.

Final options to start celery is `--pool=gevent` or `--pool=eventlet` which is what this article is finally about.Both of them are similar. But before jumping into all these details, I will give some more context about what celery was doing for me, and why I needed to explore all the options provided by celery.

## When to use celery?

- Tasks are time-consuming (e.g., sending emails, processing large data sets)
- Tasks need to be scheduled (e.g., periodic reports)
- When we need our tasks to have good and reliable retry mechanisms (eg with API calls)
- When we need to have reliable monitoring of the multiple concurrent tasks (celery provides this via [flower](https://flower.readthedocs.io/en/latest/))
- When we need to have rate limiting in our service (e.g we are making API calls)

## Why exactly did I need to use celery?

Well, I was trying to make multiple calls to OpenAI in a project using Django REST Framework. I was already pretty deep into the development of this project, having designed all my serializers, and almost all my business logic completed. Then came the problem of making concurrent calls to OpenAI. I had to think of a solution which requires less resources while also being efficient.

Having all these restrictions meant thread pool executor was automatically out of the picture(threads get expensive!). Another downside to using thread pool was the Global Interpreter Lock(GIL). The GIL only allows one thread to execute python bytecode at a time. If you want to understand GIL in detail, this is a [great resource](http://www.dabeaz.com/python/UnderstandingGIL.pdf).
So, not only was that hardware intensive, it was very limited in terms of the parallelism it offered. Threads use a shared memory, so I would have to take race conditions into consideration. I also needed to write to a Postgres DB, so the threads would unnecessarily be blocked on I/O operations.

So now, I shifted my focus to the popular AsyncIO. AsyncIO really shines when we want to have I/O intensive tasks run from a single thread. Not only I/O tasks, but anything which depends on a external factors like API calls, downloading files etc. AsyncIO uses an event loop. An event loop can be thought of as a waiter at a restaurant. It will continuously keep checking on the status of an event, if it is blocked, it will move on to the next event. However, this is an asynchronous process and Django Rest Framework is a synchronous framework. So, again i wasnt able to make use of asyncio. Sure, i could have switched to [Async Djnago Rest Framework(ADRF)](https://github.com/em1208/adrf). This is still a relatively new package, with very little support.

Overall, while Django itself has allowed async views after version 3.1, it hasnt propagated well to other parts of djnago. What I mean by this is,integral parts of Django like the Django ORM ,cache, middlewares,serializers etc. have not migrated to async.
In these, async behaviour is currently only possible to be faked via `sync_to_async`, and the maintainers of django havent really committed to making it completely async. While i could have migrated all my code to async with this `sync_to_async` decorator, it would just have added a lot of work for very little return. So, i turned to the final option: Gevent. But before going in that rabbit hole, lets talk a little bit about multiprocessing, multithreading and multitasking, terms which i have been loosely referring to this entire time

## Multiprocessing, Multithreading, Multitasking and how does it relate to concurrency?!
