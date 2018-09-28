---
layout: post
title:  "Asynchronous Python Part 1"
date:   2018-09-28 22:07:55 +0300
categories: Python
---

Second post in this series. as mentioned, our use case will be the same across the different tests:

* Synchronous ([Part 1](https://norbinsh.github.io/python/2018/09/28/asynchronous-python-part-1.html))
* Asynchronous - using asyncio and other supporting async libraries as needed (The test covered in this post) 
* Async with a mix of threads & process pools

#### What is being tested?
###### Note: Things may seem complicated for no good reason - It's on purpose to better see the performance differentiation.

We will see the completion of the following actions (as one):
* Retrieve the contents of google.com/robots.txt
* Wrap the returned result with html tags
* Open a file in write mode and save the html content inside it
* Delete the file we just created
* Repeat the process 50 times


## Test #1 - Synchronous

Gist [here](https://gist.github.com/Norbinsh/47c8ec36ea48648cf8bee3c56e3cad16)

We are breaking down (almost) every action into a function, to keep it clean & simple.

Let's start from the end, and see what our main function is calling:

```
@timeit
def main() -> None:
    url = 'https://www.google.com/robots.txt'
    ifile = "robots.html"
    for i in range(1, 50 + 1):
        print(f"Iteration number {i}")
        raw_data = retrieve_url(url)
        html_data = html_that(raw_data)
        io_save(ifile, html_data)
        clean_up(ifile)
```
\* @timeit is a decorator that will measure how long it took our calling method (main()) to finish execution.
1. `retrieve_url()` will return a response object of the HTML request
```
def retrieve_url(url: str) -> requests.models.Response:
    print(f"Retrieving URL: {url}")
    response = requests.get(url, timeout=5)
    response.raise_for_status()
    return response
```

2. `html_that()` will wrap the returned text with \<html\> tags
```
def html_that(response: requests.models.Response) -> str:
    print(f"Wrapping with <html>")
    html_content = f"""
    <html>
        <head>
            <title>
                Blocking can be bad for you.
            </title>
        </head>
        <body>
            <h1>
                Blocking can be bad for you.
            </h1>
            <p>
                {response.text}
            </p>
        </body>
    </html>
    """
    return html_content
```
3. `io_save()` will open a file in write mode, and then write the contents of the html into it
```
def io_save(input_file: str, html_page: str) -> None:
    print(f"Opening file: {input_file}")
    with open(input_file, "w") as write_file:
        write_file.write(html_page)
```
4. `clean_up()` as the name may suggest, will delete the file we create in each iteration
```
def clean_up(input_file: str) -> None:
    if os.path.exists(input_file):
        print(f"Deleting file: {input_file}")
        os.remove(input_file)
    else:
        print(f"Did not find file: {input_file}")
```

# Results
```
...
Iteration number 49
Retrieving URL: https://www.google.com/robots.txt
Wrapping with <html>
Opening file: robots.html
Deleting file: robots.html
Iteration number 50
Retrieving URL: https://www.google.com/robots.txt
Wrapping with <html>
Opening file: robots.html
Deleting file: robots.html
'main'  finished in 16.95s
```

As you can see, running this module with 50 iterations, completed without any asynchronous code in 16.95 seconds.

In the next post we will see the differences using asynchronous code where ever we can while trying to keep the code
as close to the original as possible.
