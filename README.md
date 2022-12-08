# How to format strings for logging in Python
## Lightning talk at PyData 2022
This is the written version of a lightning talk I gave at PyData 2022. It also includes a
few examples that I had to cut out of the talk for time reasons. Large parts of the talk
are based on discussions of this topic by others, so I have included links to sources here
wherever possible. Especially [this blog post by Izabbela
Kowal](https://dev.to/izabelakowal/what-is-the-best-string-formatting-technique-for-logging-in-python-d1d)
provides an excellent overview of the whole topic.

## Intro
The question "how to format strings for logging" seems like a very random topic and also a
rather unimportant detail, but as often, the more you dig into a topic, the more curious
things you discover. Also I felt like the scope of this was small enough for a lightning
talk.

## Recap: different ways to format strings
Python has different ways to achieve basically the same thing:
```python
>>> name = 'World'
>>> 'Hello, %s' % name
'Hello, World'

>>> from string import Template
>>> t = Template('Hello, $name')
>>> t.substitute(name=name)
'Hello, World'

# since Python 3.0
>>> 'Hello, {0}'.format(name)
'Hello, World'

# since Python 3.6
>>> f'Hello, {name}'
'Hello, World'
```

## The logging module
The logging module has again a different notation:
```python
>>> import logging
>>> logging.basicConfig()
>>> logger = logging.getLogger(__name__)
>>> logger.setLevel(logging.WARNING)

>>> task_name = 'MyTask'
>>> logger.info('Starting task: %s', task_name)
```
Note that this is different from the % notation in the previous section, because here we
pass the variable as a parameter to the `logger.info` method.

An alternative would be to use f-strings like so:
```python
>>> logger.info(f'Starting task: {task_name}')
```
However, I have noticed that many people still seem to use the first formatting style,
even with f-strings being around for quite a while now.

So the initial question that led me to this talk was: Is there actually a good reason
**not** to use f-strings in this case?

The reason people usually give is that, when using the first notation, the logging module
does a kind of "lazy evaluation": it only formats the string if it acutally has to be
printed. Notice that the logging level in the example is set to `WARNING`. So the calls
to logging.info will not produce output. The logging module realizes that and avoids
formatting the string, if you use the first formatting style. In contrast, the f-string is
evaluated immediately, even before it is passed to `logger.info`. Therefore, goes the
argument, the `%s` formatting style saves you some computation time.

Let's look at this in more detail.

## Speed
There is a [tweet by Python developer Raymond
Hettinger](https://twitter.com/raymondh/status/1205969258800275456) comparing the
different ways of string formatting. f-strings turn out to be the fastest. So there is a
problem: the benefit that you get from %s formatting in case the message is not printed is
counterbalanced by the better performance of f-strings in case the message *is* printed.
And at the time of writing your application code, you don't know what the logging level
will be later on.

I did some basic timing experiments (using Jupyter magic) with both formatting styles and
different log levels:

```python
res = %timeit -o logger.info('Starting task: %s at time: %d', task_name, now)
print(res)
991 µs ± 79.6 µs per loop (mean ± std. dev. of 7 runs, 1,000 loops each)

res2 = %timeit -o logger.info(f'Starting task: {task_name} at time: {now}')
print(res2)
896 µs ± 69.1 µs per loop (mean ± std. dev. of 7 runs, 1,000 loops each)
```
The logging level is set to `INFO` in these examples, so the messages above are printed.
One can see that f-strings are a little faster, although the difference is quite small.

The following example is the same except that `logger.debug` is now called, so the
messages are not printed:

```python
%%timeit
logger.debug('Starting task: %s at time: %d', task_name, now)
286 ns ± 14.6 ns per loop (mean ± std. dev. of 7 runs, 1,000,000 loops each)

%%timeit
logger.debug(f'Starting task: {task_name} at time: {now}')
399 ns ± 21.8 ns per loop (mean ± std. dev. of 7 runs, 1,000,000 loops each)
```
The first thing to note is that the absolute times are much smaller when
no output is generated: we are in the nanosecond range here, as compared to the
microsecond range above.

Also, the *relative* difference between the two formatting styles is larger here (around
40%), so `logging`'s "lazy evaluation" behavior does pay off. However, the *absolute*
difference is much smaller (around 100 ns, compared to 100 µs above).

What can we conclude from this? First, when optimizing your application, you probably care
more about absolute speed gains, which would imply a tendency to use f-strings rather
than %s formatting. But my actual point is that these differences are so small that they
do not have any practical relevance for the vast majority of use-cases (if you know an
example where such differences do matter, I'd love to hear about it).

## Security considerations
There are some interesting things to be discovered here. Especially `str.format` and
f-strings are more powerful than people may realize at first. For example, you are
probably familiar with this:
```python
>>> 'The answer is {0}'.format(42)
'The answer is 42'
```
But did you know that you can access attributes and methods of the passed object, too?:
```python
>>> 'The answer is {0.__class__}'.format(42)
"The answer is <class 'int'>"
```
This can be used to extract potentially sensitive information:
### Extracting information
[Armin Ronacher](https://lucumr.pocoo.org/2016/12/29/careful-with-str-format/) and
[Podalirius](https://podalirius.net/en/articles/python-format-string-vulnerabilities/)
have pointed this out in blog posts. Here is a slightly simplified version of their
examples:
```python
CONFIG = {
    'API_KEY': '7a2cdf4280a279ffb968e267ab8e9a0e809d0f3d'
}

class Event:
    def __init__(self):
        pass

event = Event()
print('{event.__init__.__globals__[CONFIG][API_KEY]}'.format(event=event))
# prints: 7a2cdf4280a279ffb968e267ab8e9a0e809d0f3d
```
### String padding attack
A different kind of attack is using format strings to add a huge amount of whitespace to a
printed message, causing the Python program to freeze and potentially crash or run out of
memory.

This notation pads the string "hello" with leading whitespace so that it is 9 characters
long in total:
```python
>>> '{:>9}'.format('hello')
'    hello'
```

However, [the number can be arbitrarily
large](https://stackoverflow.com/questions/15356649/can-pythons-string-format-be-made-safe-for-untrusted-format-strings):
```python
>>> '{:>999999999}'.format('hello')
# freezes / crashes
```

[The same thing can be achieved using the logging
syntax](https://peps.python.org/pep-0675/#logging-format-string-injection), provided that
you use a named variable in the format string and pass a dictionary as as a parameter:
```python
>>> logger.info('%(foo)999999999s', {'foo': 'hello'})
# freezes / crashes
```

### What about f-strings?
The point I want to make with these examples is that security becomes an issue once the
format string comes from an untrusted source. You could imagine that an application wants
to let the user control the formatting of log messages, so it reads the format string e.g.
from a configuration file. A malicious user could exploit that.

I would argue that f-strings are somewhat safe against these kind of attacks, because they
are evaluated immediately. So the situation that the program reads an unformatted string
with unknown content and then later formats it is not possible with f-strings, [provided
that you do not mix them with `%s` style formatting](https://bugs.python.org/issue46200).
(Of course, both attacks theoretically also work with f-strings.)

## So which one should I use?
- In terms of speed: it (almost always) doesn't matter! The differences are just too tiny.
- [Readability counts](https://peps.python.org/pep-0020/). In my opinion f-strings are the
most readable of all methods.
- Think about security if you let users control the format string.

## Caveats
```python
# Don't do this
>>> logger.info('The result is: %s', expensive_function())

# Neither this
>>> logger.info(f'The result is: {expensive_function()}')
```
The logging documentation [provides a workaround for this
situation](https://docs.python.org/3/howto/logging.html#optimization), however I would
argue that this is a very unnatural thing to do anyways. If you're running an
`expensive_function()`, you probably want to do something more with its result than just
log it, e.g. write it to a file, show it on a web page, or use it further in your program.

## Changing logging's formatting style
When the logging module has to format a string, it internally uses the `'string' % args`
syntax
([Source](https://github.com/python/cpython/blob/main/Lib/logging/__init__.py#L392)).
This happens in a `LogRecord` class [which can be
subclassed](https://docs.python.org/3/howto/logging-cookbook.html#using-particular-formatting-styles-throughout-your-application)
in order to override this behavior with the formatting style of your choice.

However, alternatives like `string.Template` and `str.format` would have worse performance
than %s formatting (according to Raymond Hettinger's tweet above), and f-strings cannot be
lazily evaluated.

## Log aggregators as a special case
[Izabela](https://dev.to/izabelakowal/what-is-the-best-string-formatting-technique-for-logging-in-python-d1d)
and [Vitaly](https://blog.pilosus.org/posts/2020/01/24/python-f-strings-in-logging/)
mention that the formatting style makes a difference when you use a log aggregator like
Sentry. In this case %s formatting is preferable because it enables proper grouping of the
logging messages in Sentry.
