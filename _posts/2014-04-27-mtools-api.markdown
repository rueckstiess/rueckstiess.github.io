---
layout: post
title: mtools Python API
categories: mtools
---

A less known fact about [_mtools_][mtools] is that in addition to the command line scripts, _mtools_ also offers a Python API with many of the basic elements of log line parsing that can be called from your own Python scripts. The classes are not publicly documented yet so the term "API" might be a little strong, but they have been around for a few releases now and could be useful if the scripts in _mtools_ aren't exactly what you are looking for, but you don't want to start from scratch. 

In this article I'm going to explain two very useful classes when working with MongoDB log files: The `LogFile` and `LogEvent` classes. Follow-up posts will cover some other classes that _mtools_ uses internally and might be useful for your own scripts.

### The LogFile class

The first class I want to talk about is `LogFile`, located under [mtools/util/logfile.py](https://github.com/rueckstiess/mtools/blob/master/mtools/util/logfile.py). The `LogFile` class is a generic wrapper for a log file, and offers utility methods around log _files_ (in contrast to individual log _lines_), for example easy access to general properties like start and end date of the file, the file size and the number of lines, but also more MongoDB-specific things like the times of server restarts, what version(s) can be found in the file (if any) or if the log file was created by a mongod or mongos. Here some examples to access these properties:

~~~python
from mtools.util.logfile import LogFile

# LogFile takes a file stream object
logfile = LogFile( open('./mongod.log', 'r') )

print "info about log file %s" % logfile.name
print "log file reaches from %s to %s" % (logfile.start, logfile.end)
print "it has %i lines" % logfile.num_lines
~~~

This would output something like: 

~~~
info about logfile ./mongod.log
logfile reaches from 2014-04-05 06:39:57+00:00 to 2014-04-06 06:27:07+00:00
it has 58224 lines
~~~

The output isn't very pretty, but it's just a simple example and the `.start` and `.end` properties return `datetime` objects which are flexible to deal with and easily formattable. 

In addition, the `LogFile` class acts as an iterator over the log lines, returning `LogEvent` objects for each line. We will see an application of this in the `LogEvent` examples below.


### The LogEvent class

The `LogEvent` class in [mtools/util/logevent.py](https://github.com/rueckstiess/mtools/blob/master/mtools/util/logevent.py) covers each individual log line in a file, and the easiest way to get the `LogEvent` objects is to iterate over a file, as mentioned above. To continue the example from before, you can do:

~~~python
for logevent in logfile:
    print logevent
~~~

This would print out the entire log file, line by line. This is because the `LogEvent` class has its own implementation of `__str__`, printing out the actual log line string, instead of something less useful like `<mtools.util.logevent.LogEvent object at 0x103475690>`, which would be the standard Python string representation of an object.

Another way of initializing a `LogEvent` object is by simply providing the entire log line string in the constructor.  Here an example, wrapping the line in `"""` triple-quote strings to avoid conflicts with the quotes inside the log line:

```python
>>> from mtools.util.logevent import LogEvent
>>> logevent = LogEvent("""Sun Apr  6 06:26:41 [conn9486] query serverside.user query: { _types: "User", emails.email: thomas@10gen.com } ntoreturn:1 ntoskip:0 nscanned:11538 keyUpdates:0 locks(micros) r:106712 nreturned:1 reslen:1543 106ms""")
```


The `LogEvent` class offers a lot of properties and methods that are useful. The properties are evaluated lazily on first call, to not waste any time computing something that isn't necessary. Every `LogEvent` has a `.datetime` property that returns a datetime object (except in the case when the log line didn't have a timestamp at the start, which should not usually happen). Other useful properties of `LogEvent` objects are:

`.line_str`
:    The original string of the log line, newline characters stripped

`.split_tokens`
:    The line string tokenized on whitespace

`.duration`
:    if the event had a duration, return the time in milliseconds it took to complete. Also works with _mmap flushes_.

`.thread`
: The thread name, usually `conn#####` where `#####` is a number, but can also be something like `rsMgr`. In the log line, this is printed between `[` and `]` after the timestamp.

`.operation`
: if the event was an operation, returns what the operation was. One of `query`, `getmore`, `remove`, `update`, 
`command`.

`.pattern`
: if the event was a `query`, `getmore`, `update` or `remove`, this returns the query pattern. For example, if the query was on `{a: "foo", b: 42}`, the pattern would be `{a: 1, b: 1}`.

Below example shows some of the properties that have been extracted from the log line in the example before:

```python
>>> logevent.datetime
datetime.datetime(2014, 4, 6, 6, 26, 41, tzinfo=tzutc())

>>> logevent.duration
106

>>> logevent.pattern
'{"_types": 1, "emails.email": 1}'

>>> logevent.thread
'conn9486'
```

The class supports a lot more of these properties, for example "n-counters", like `nscanned`, `nreturned`, etc. where they apply. Finally, `LogEvent` also has the ability to output the results as a Python dictionary or in JSON format, with the `.to_dict()` and `.to_json()` methods.


[mtools]: https://github.com/rueckstiess/mtools
