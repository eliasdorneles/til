---
title: "How to add extra log context with loguru"
date: "2023-11-07T19:50:42+01:00"
tags:
  - python
  - loguru
  - logging
---

[Loguru](https://pypi.org/project/loguru/) is a Python library that provides a
neat logging API, and I recently learned [how to add extra context to the logs](https://loguru.readthedocs.io/en/stable/overview.html#structured-logging-as-needed).

This is really nice when you're using structured logs, because you can do all
sorts of things based on the data you collect like this.

There are mainly two options:

## 1) Use `logger.bind(**kwargs)` to create a local logger with the context information

Example:

```python
from loguru import logger

def some_func1():
    ctx_logger = logger.bind(func_name='some_func1')
    ctx_logger.info('Hello there!')

def some_func2():
    ctx_logger = logger.bind(func_name='some_func2')
    ctx_logger.info('Hiya!')
    ctx_logger.info('Nice to see you here!')


# for testing, let's add an extra handler that displays the extra context:
import sys
logger.add(sys.stderr, format="{message} | {extra}")

some_func1()
some_func2()
some_func1()
```

Running the above example, you will see:

```
2023-11-07 20:06:37.883 | INFO     | __main__:some_func1:3 - Hello there!
Hello there! | {'func_name': 'some_func1'}
>>> some_func2()
2023-11-07 20:06:37.883 | INFO     | __main__:some_func2:3 - Hiya!
Hiya! | {'func_name': 'some_func2'}
2023-11-07 20:06:37.883 | INFO     | __main__:some_func2:4 - Nice to see you here!
Nice to see you here! | {'func_name': 'some_func2'}
>>> some_func1()
2023-11-07 20:06:37.884 | INFO     | __main__:some_func1:3 - Hello there!
Hello there! | {'func_name': 'some_func1'}
```

> In the above output, we see each line logged in double: the first time by the
> default handler/formatter, the second one by the custom one we added which
> prints the extra context.

## 2) Use `logger.contextualize(**kwargs)` to add information to the context-local state

By context-local state here, I mean in the sense of a [context
variable](https://docs.python.org/3/library/contextvars.html), which will be
the equivalent of a using a thread-local variable or an async-context-local in
the case of asynchronous code.

This is useful in a web application (Django, FastAPI, etc) to add information
to the logs in the context of a request being handled.

It's as easy as:

```python
from loguru import logger

def some_func1():
    logger.info('Hello there!')

def some_func2():
    logger.info('Hiya!')
    logger.info('Nice to see you here!')


# for testing, let's add an extra handler that displays the extra context:
import sys
logger.add(sys.stderr, format="{message} | {extra}")

with logger.contextualize(task_id=1234):
    some_func1()
    some_func2()
```

The output for the above example will be:

```
2023-11-07 20:06:24.980 | INFO     | __main__:some_func1:2 - Hello there!
Hello there! | {'task_id': 1234}
2023-11-07 20:06:24.980 | INFO     | __main__:some_func2:2 - Hiya!
Hiya! | {'task_id': 1234}
2023-11-07 20:06:24.981 | INFO     | __main__:some_func2:3 - Nice to see you here!
Nice to see you here! | {'task_id': 1234}
```
