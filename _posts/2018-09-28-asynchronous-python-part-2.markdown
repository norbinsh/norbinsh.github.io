---
layout: post
title:  "Asynchronous Python Part 2"
date:   2018-09-28 22:07:55 +0300
categories: Python
---

Second post in this series. as mentioned, our use case will be the same across the different tests:

* Synchronous ([Part 1](https://norbinsh.github.io/python/2018/09/28/asynchronous-python-part-1.html))
* Asynchronous - using asyncio and other supporting async libraries as needed (The test covered in this post) 
* Comparison with go-lang's 'goroutines' / 'channels'

#### What is being tested?
###### Note: Things may seem complicated for no good reason - It's on purpose to better see the performance differentiation.

We will see the completion of the following actions (as one):
* Retrieve the contents of google.com/robots.txt
* Wrap the returned result with html tags
* Open a file in write mode and save the html content inside it
* Delete the file we just created
* Repeat the process 50 times


## Test #2 - Asynchronous

Gist [here](https://gist.github.com/Norbinsh/02c398e9e1d6d0ce1a9031aefc0e6050)

### So what did we change?

* Got rid of `'requests'` in favour of  `'aiohttp'` which is async supported out of the box.
* Hello coroutines! Started using asyncio APIs (async / await) where possible 

Remember that in general there are 3 types of awaitables in general - coroutines, tasks, and futures.


```
loop = asyncio.get_event_loop()
```

Here we create the event loop - this is where the tasks runs coroutines.


```
loop.run_until_complete(orchestrator())
```

When a task awaits completion, the event loop knows that and schedule to run other stuff in the meanwhile.
Notice that we created a special function `"orchestrator()"` (below) that will actually create the tasks - we then run
it until completion.

```
async def orchestrator():

    url = 'https://www.google.com/robots.txt'
    ifile = "robots.html"

    tasks = []
    for _ in range(1, 50 + 1):
        tasks.append(asyncio.create_task(retrieve_url(url)))
    for task in tasks:
        resp = await task
        html_content = await html_that(resp)
        await io_save(ifile, html_content)
        await clean_up(ifile)
```

We are using the `create_task()` to wrap the coroutine into a task and schedule execution, then - we await on each of 
the tasks to finish.

# Results
```
...
Deleting file: robots.html
Wrapping with <html>
Opening file: robots.html
Deleting file: robots.html
'main'  finished in 0.53s
```
*0.53* seconds! that's 32 times faster with the cost of very minor added complexity.

In the next post we will try to spice things up using threads and process pool executors, in addition to asyncio.
