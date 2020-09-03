---
title: "Logging In Python"
date: 2020-09-03T9:00:00+05:30
draft: false
---

Print is a simple handy function that we use alot but that is not enough when it comes to debugging. 

1. When an error occur in production you may prefer to send a mail or post that into slack channel etc for monitoring than just logging into a file. 

2. And another import thing is that you need to know the state of the application when an error occured, so that you can reproduce and fix the error. 

Logging is a handy tool to do both of these tasks easily. You need to use logging functions like logging.info, logging.error etc to log the message. Logging functions are named after the level or severity of the events they are used to track.

By doing this, messages are grouped by severity and you get a chance to handle every group in a different manner. logging module provides variety of handlers to do that.

#### Setup Logger For A Module

Let us see an example. Log all the messages into debug.log and log only severity level error and above messages into error.log.

``` python
"""
* Log all the messages into debug.log
* Log only severity level error and above messages into error.log
"""

import logging
from logging import FileHandler

# create logger
logger = logging.getLogger(__name__)
logger.setLevel(logging.DEBUG) # handles messages of severity DEBUG and above

# create a handler to log all messages of sevirity DEBUG and above.
debug_handler = FileHandler("debug.log")
debug_handler.setLevel(logging.DEBUG)

# create a handler to log all messages of sevirity ERROR and above.
error_handler = FileHandler("error.log")
error_handler.setLevel(logging.ERROR)


# add handlers to logger.
# logger sends all the incoming messages to all the handlers.
logger.addHandler(debug_handler)
logger.addHandler(error_handler)

def division(a, b):
    logger.debug(f"{a} divided by {b}")  # this message comes only in debug.log
    try:
        return a/b
    except ZeroDivisionError:
        logger.error("Zero division error", exc_info=True) # this message comes in both error.log & debug.log


if __name__ == "__main__":
    division(4, 2)
    division(4, 0)
```
Here is the [gist link](https://gist.github.com/leela/395dab9b502f517916a008f2dded12bb) for the same. Hope the code is self explanatory, here are the takeaway points.

* You can create a logger and attach handlers to the logger.
* You can set the entry level for loggers and handlers by using setLevel.
* passing below entry level messages will be ignored. Ex: error_handler in code ignore the debug message that logger passed to it
* Best practice is creating logger using `logging.getLogger(__name__)`, when we do this we know source of the log message from the logger name.

#### Setup Logger For A Package
In the previous example we have created a logger and added handlers to it. What if we have a package, are we going to create separate logger for every module using `logging.getLogger(__name__)` and add handlers to all those loggers :scream:?

Though you can create logger with what ever the name you want and can use same logger all around, the preferred way is to use module level loggers. And do not worry, you **do not need to add handlers to all the loggers**. Let me clarify that using Covid analogy.

If we format our addresses using dot seperator, it looks something like `country.state.distict.mandal.village.home`.
every address has a parent and parent has its own address. Ex: district address is `country.state.distict`.
what about `country` address? It is part of world(Though not mentioned). 

How covid spreads? spreads from home to the village, from village to mandal, from mandal to district, from distict to state goes on. If you see carrier or handler is the human.

Logger names are like our home addresses separated by dots. log message is like virus. It is passed to all the handlers of that logger (like covid comes to people in the house). from there it goes to parent handlers, from there goes to its parent handlers and goes on. Like `world` we have default logger called `Root`. like virus propagating till world, message propagate till `Root` logger (Though from there covid spread to other countries, log message stops at root logger handlers).

What is the advantage we get when message propagate till the root logger? As the same message is going to all the handlers in the hierarchy, You can add handlers to the parent in a hierarchy and relax. **A common scenario is to attach handlers only to the root logger, and to let propgation take of the rest.**


Let us see an example. for a package, log all the messages into debug.log and log only severity level error and above messages into error.log.

**Here is my package structure**
``` bash
logging_example
    > __init__.py
    > add.py
    > div.py

```

**logging_example/__init__.py**
``` python
import logging
from logging import FileHandler
from .add import addition
from .div import division

root_logger = logging.getLogger()   # Returns the root logger
root_logger.setLevel(logging.DEBUG) # handles messages of severity DEBUG and above

# create a handler to log all messages of sevirity DEBUG and above.
debug_handler = FileHandler("debug.log")
debug_handler.setLevel(logging.DEBUG)
# We can pass the  message format using setFormatter. Notice that log name is added.
debug_handler.setFormatter(logging.Formatter('%(name)s::%(message)s'))

# create a handler to log all messages of sevirity ERROR and above.
error_handler = FileHandler("error.log")
error_handler.setLevel(logging.ERROR)
# every record in log looks like `logger_name::log_message`
error_handler.setFormatter(logging.Formatter('%(name)s::%(message)s'))

# add handlers to root logger.
# logger sends all the incoming messages to all the handlers.
root_logger.addHandler(debug_handler)
root_logger.addHandler(error_handler)
```

logging_example/add.py
``` python
import logging

logger = logging.getLogger(__name__)

def addition(a, b):
    logger.debug(f"Adding {a} and {b}")  # this message comes only in debug.log
    return a+b
```

logging_example/div.py
``` python
import logging

logger = logging.getLogger(__name__)

def division(a, b):
    logger.debug(f"{a} divided by {b}")  # this message comes only in debug.log
    try:
        return a/b
    except ZeroDivisionError:
        logger.error("Zero division error", exc_info=True) # this message comes in both error.log & debug.log
```
Here is the [github link](https://github.com/leela/logging-example.git) to clone the package. 

* In `__init__.py`, `getLogger` returns **root** logger as it is called without arguments.
* Though we have multiple loggers, we have added handlers only to a **root** logger
* root logger log messages through propagation.
* You can use `SetFormatter` to log the message in specified format.

``` shell
>>> import logging_example  # import the package
>>> import logging

>>> logging.root   # Root logger
<RootLogger root (DEBUG)>

# List of loggers present in the system. you can see that system added a `logging_example` logger.
>>> logging.root.manager.loggerDict
{'logging_example.add': <Logger logging_example.add (DEBUG)>, 'logging_example': <logging.PlaceHolder object at 0x10eee19d0>, 'logging_example.div': <Logger logging_example.div (DEBUG)>}

# This gets me the logger named `logging_example.add`.
# getLogger creates new logger only when there is no logger found with the given name.
>>> logger = logging.getLogger('logging_example.add')

>>> logger.handlers   # We have not added any handlers to logger `logging_example.add`
[]
>>> logger.root.handlers  # We have added handlers to `root` logger
[<FileHandler /Users/leela/logging-example/debug.log (DEBUG)>, <FileHandler /Users/leela/logging-example/error.log (ERROR)>]
>>> 
```

* I get same behaviour even If I add handlers to `logging_example` logger instead of `root`. as that is the parent to both `logging_example.add` and `logging_example.div` loggers.
* Set the handlers to parent loggers and let the propagation does the rest.
* Logging modules provide a way to stop propagation incase you need it.

Hope this is clear. With this basic understanding we can easily setup loggers for any project. We will see how to setup logger for flask application in next post.