---
layout: post
title: mtools Python API
categories: mtools
---

A less known fact about _mtools_ is that in addition to the command line scripts, _mtools_ also offers a Python API with many of the basic elements of log line parsing that can be called from your own Python scripts. The classes are not publicly documented yet so the term "API" might be a little strong, but they have been around for a few releases now and could be useful if the scripts in _mtools_ aren't exactly what you are looking for. 

In this article I want to introduce some of the more interesting and useful mtools classes.

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

The `LogEvent` class offers a lot of properties and methods that are useful. The properties are evaluated lazily on first call, to not waste any time computing something that isn't necessary. Every `LogEvent` has a `.datetime` property that returns a datetime object. Other properties that are always present are:


